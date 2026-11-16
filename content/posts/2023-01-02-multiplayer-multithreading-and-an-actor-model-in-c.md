---
title: "Multiplayer, multithreading, and an actor model in C++"
date: "2023-01-02T14:51:43Z"
description: "Warmonger Dynasty Devlog #2: Multiplayer and Networking"
tags: ["gamedev", "programming", "networking", "cpp", "multiplayer"]
author: "David Delassus"
---

I’m currently writing a 4X strategy game in C\+\+ with no engine. I already talked a little about it in [my first devlog](/posts/2022-12-25-im-embedding-my-game-assets-into-the-executable-here-s-how-i-did-it).

During previous gamedev experiments, I learned that if you do not plan early for multiplayer, you’ll have a hard time refactoring the code. This is why networking was on my top priority list. Especially since I’m making the game from scratch, with no help from Unity, or Unreal Engine, or whatever. Battle-tested engines provide lots of tools to ease the pain of making games, but hey! where would be the fun? 🙂

---

## Dedicated server or builtin?

Those are basically the 2 options you have. Your server can either be its own executable, separated from your game, which can then be run on a dedicated server (baremetal, in the cloud, …). Or you can embed it in your game, which will act both as a server and a client.

I went with the second option because it fits into my goal of distributing everything in a single executable.

Even with the second option, there is two ways to achieve this, either with a process, or with a thread.

A thread allows me to share memory between the server thread and the main thread, this enables the introduction of synchronisation points through `std::promise<T>`. This is the solution I went with.

The server is responsible for creating and managing the game state, and relay changes to it to the clients via a TCP socket (latency is not a problem since it’s a turn-based game, and not a real-time one).

When started, the server thread will create an authentication token that will be used by the player hosting the game to authenticate admin requests:

- addind an AI client
- kicking a player
- start the game

Here is the skeleton of the `server_thread` class I’m gonna use:

```cpp
#pragma once

#include <future>
#include <thread>
#include <string>
#include <vector>
#include <memory>
#include <optional>

#include <SDL_net.h>          // for a simple portable abstraction over TCP sockets
#include <nlohmann/json.hpp>  // to encode the messages exchanged via the TCP socket

#include <game/common/state.hpp>     // defines the game state structures
#include <game/common/settings.hpp>  // defines the game settings structures
#include <game/common/auth.hpp>      // defines the auth_token type

namespace game::server {
  struct client {
    TCPsocket socket;
    IPaddress *peer;
  };

  class server_thread {
    private:
      std::unique_ptr<common::settings> m_game_settings;
      std::unique_ptr<common::game_state> m_game_state{nullptr};
      bool m_in_lobby{true};
      common::auth_token m_auth_token;

      std::thread m_thread;
      bool m_running;
      std::promise<common::auth_token> m_ready_promise;
      std::shared_future<common::auth_token> m_ready_state;
      std::promise<void> m_stop_promise;
      std::shared_future<void> m_stop_state;

      TCPsocket m_server_socket;
      SDLNet_SocketSet m_socket_set;
      std::vector<client> m_clients;

    public:
      server_thread(const common::settings &game_settings);
      ~server_thread();

      std::shared_future<common::auth_token> wait_ready();
      std::shared_future<void> wait_stopped();
      std::shared_future<void> stop();

    private:
      void run();

      // ...
  };
}
```

Here, we have 2 promises: `m_ready_promise` and `m_stop_promise`. They are used to inform the main thread of the server’s lifecycle. Since the main thread is gonna call the `wait_ready()` and `wait_stopped()` methods multiple times, we need to return a `std::shared_future<T>` which allows the caller to get the future’s result multiple times.

Now for the implementation:

```cpp
#include <exception>

#include <fmt/format.h>

#include <game/server/thread.hpp>

namespace game::server {
  server_thread::server_thread(const common::settings &game_settings)
    : m_game_settings(std::make_unique<common::settings>(game_settings),
      m_thread(&server_thread::run, this),
      m_ready_state(m_ready_promise.get_future().share()),
      m_stop_state(m_stop_promise.get_future().share())
  {}

  server_thread::~server_thread() {
    m_running = false;
    m_thread.join();
  }

  std::shared_future<common::auth_token> server_thread::wait_ready() {
    return m_ready_state;
  }

  std::shared_future<void> server_thread::wait_stopped() {
    return m_stop_state;
  }

  std::shared_future<void> server_thread::stop() {
    m_running = false;
    return m_stop_state;
  }

  void server_thread::run() {
    // setup TCP socket server
    // ...
    if (m_server_socket == nullptr) {
      auto error = fmt::format("Could not create server socket: {0}", SDLNet_GetError());
      // ...
      m_ready_promise.set_exception(
        std::make_exception_ptr(std::runtime_error(error))
      );
      m_stop_promise.set_value();
      return;
    }

    // setup game state
    // generate auth token

    m_ready_promise.set_value(m_auth_token);

    try {
      m_running = true;

      while (m_running) {
        // wait for data to be available on server/client sockets
        // accept/reject new connections
        // process client requests
      }

      m_stop_promise.set_value();
    }
    catch (...) {
      m_stop_promise.set_exception(std::current_exception());
    }

    // free resources
  }
}
```

Thanks to those promises, the main thread can display a loading screen while the “ready future” is not completed. And every frame, we can check the status of the “stop future” to detect and handle errors or unexpected terminations.

The main thread will create the server thread like this:

```rust
// ...

auto server_fut = std::async(std::launch::async, [&]() {
  m_server = std::make_shared<server::server_thread>(m_game_settings);

  try {
    m_auth_token = m_server->wait_ready().get();
    m_creation_error = std::nullopt;
  }
  catch (const std::exception &exc) {
    std::string err = exc.what();
    m_creation_error = err;
  }
});

// ...
```

Every frame, we will check the status of the `server_fut` variable. As long as it’s not completed, we’ll display the loading screen, and if it’s ready, the value of `m_creation_error` will indicate if the creation was successful. If it was, we can go to the lobby. If it wasn’t, we can display the error message and go back to the title screen. Here is how it looks:

```cpp
#include <chrono>

using namespace std::chrono_literals;

// ...

if (server_fut.wait_for(0s) == std::future_status::ready) {
  if (m_creation_error.has_value()) {
    // display error message
  }
  else {
    // go to lobby
  }
}
else {
  // display loading screen
}

// ...
```

We also save the auth token returned by the “ready future”, for later use in the lobby.

**NB:** The server thread is created in a shared pointer because it will be passed around to the next game scenes (more about this in a later devlog).

---

## A message queue for the client thread

The client will be run in a thread as well, every player who connects to a game will start this thread upon entry in the lobby.

The client thread is responsible to interact with the server via the TCP socket, it will receive the game state changes and other notifications.

The main thread will then query the client thread to display in the UI or sync the entities and their components with the latest data.

This means the client will have a queue where the main thread can publish the queries. Every message in the queue will hold an `std::promise<T>` that will be resolved by the client thread.

That queue needs to be thread-safe, not a complicated endeavor in C\+\+20:

```cpp
#pragma once

#include <mutex>
#include <queue>
#include <optional>


template <typename T>
class thqueue {
  public:
    void push(const T &data) {
      std::lock_guard<std::mutex> lock(m_mutex);
      m_queue.push(data);
    }

    std::optional<T> pop() {
      std::unique_lock<std::mutex> lock(m_mutex);

      if (m_queue.empty()) {
        return std::nullopt;
      }
      else {
        auto data = m_queue.front();
        m_queue.pop();
        return data;
      }
    }

  private:
    std::queue<T> m_queue;
    std::mutex m_mutex;
};
```

This queue can hold only one type of message, which is fine, we can use `std::variant` to hold our different message types:

```cpp
#pragma once

#include <future>
#include <variant>
#include <memory>

// ...

namespace game::client {
  namespace client_queries {
    struct disconnect {
      std::promise<void> promise;
    };

    struct set_ready {
      std::promise<void> promise;
    };

    struct unset_ready {
      std::promise<void> promise;
    };

    struct kick_player {
      common::auth_token auth_token;
      common_player_id_type player_id;
      std::promise<void> promise;
    };

    struct get_game_settings {
      std::promise<std::optional<common::settings>> promise;
    };

    // ...
  }

  using client_query = std::variant<
    std::shared_ptr<client_queries::disconnect>,
    std::shared_ptr<client_queries::set_ready>,
    std::shared_ptr<client_queries::unset_ready>,
    std::shared_ptr<client_queries::kick_player>,
    std::shared_ptr<client_queries::get_game_settings>,
    // ...
  >;
}
```

**NB:** We’re using a shared pointer to make it easier to pass around the promise within the structure. There might be a better idea here, but it works.

Now that our queries are defined, we can write the interface:

```cpp
#pragma once

#include <future>
#include <thread>
#include <string>

#include <nlohmann/json.hpp>
#include <SDL_net.h>

#include <game/common/thqueue.hpp>
#include <game/client/queries.hpp>

namespace game::client {
  class client_thread {
    private:
      std::thread m_thread;
      bool m_running;
      std::promise<common::player_id_type> m_ready_promise;
      std::shared_future<common::player_id_type> m_ready_state;
      std::promise<void> m_stop_promise;
      std::shared_future<void> m_stop_state;

      TCPsocket m_socket;
      SDLNet_SocketSet m_socket_set;

      thqueue<client_query> m_query_queue;

    public:
      client_thread(const std::string &host, int port);
      ~client_thread();

      std::shared_future<common::player_id_type> wait_ready();
      std::shared_future<void> wait_stopped();
      std::shared_future<void> stop();

      std::future<void> query_disconnect();
      std::future<void> query_set_ready();
      std::future<void> query_unset_ready();
      std::future<void> query_kick_player(common::auth_token auth_token, common::player_id_type player_id);
      std::future<std::optional<common::settings>> query_get_game_settings();
      // ...

    private:
      void run(const std::string &host, int port);

      // ...

      void handle_queries();

      void handle_query(std::shared_ptr<client_queries::disconnect> query);
      void handle_query(std::shared_ptr<client_queries::set_ready> query);
      void handle_query(std::shared_ptr<client_queries::unset_ready> query);
      void handle_query(std::shared_ptr<client_queries::kick_player> query);
      void handle_query(std::shared_ptr<client_queries::get_game_settings> query);
      // ...
  };
}
```

Similar to the server thread, the client thread will have a “ready” and a “stop” promises which informs the main thread about the client’s lifecycle.

Then, we have a bunch of `query_*()` methods which are in charge of creating the query structure (containing the promise) and then returning the future so that the main thread can wait for the response.

Finally, we have a bunch of `handle_query()` methods which are in charge of processing the query, and resolving its promise.

The implementation of the thread’s loop is very similar to the server thread:

```cpp
#include <game/client/thread.hpp>

namespace game::client {
  client_thread::client_thread(const std::string &host, int port)
    : m_thread(&client_thread::run, this, host, port),
      m_ready_state(m_ready_promise.get_future().share()),
      m_stop_state(m_stop_promise.get_future().share())
  {}

  client_thread::~client_thread() {
    m_running = false;
    m_thread.join();
  }

  std::shared_future<common::player_id_type> client_thread::wait_ready() {
    return m_ready_state;
  }

  std::shared_future<void> client_thread::wait_stopped() {
    return m_stop_state;
  }

  std::shared_future<void> client_thread::stop() {
    m_running = false;
    return m_stop_state;
  }

  void client_thread::run(const std::string &host, int port) {
    // create socket

    try {
      m_running = true;

      while (m_running) {
        // wait and receive data from the socket
        // the ready promise is set once the server gave us our player_id

        handle_queries();
      }

      // notify the server we are disconnecting

      m_stop_promise.set_value();
    }
    catch (...) {
      m_stop_promise.set_exception(std::current_exception());
    }

    // empty the message queue to avoid blocking the main thread
    handle_queries();

    // free socket
  }

  void client_thread::handle_queries() {
    bool not_empty = true;

    while (not_empty) {
      auto query = m_query_queue.pop();

      if (query.has_value()) {
        std::visit(
          [&](auto &&query_data) { handle_query(query_data); },
          query.value()
        );
      }
      else {
        not_empty = false;
      }
    }
  }

  // ...
}
```

Here, `std::visit` allows us to dispatch the query to the correct handler without much hassle. Thank you C\+\+20!

Now, we can implement our queries:

```cpp
std::future<void> client_thread::query_disconnect() {
  auto query = std::make_shared<client_queries::disconnect>();
  m_query_queue.push(query);
  return query->promise.get_future();
}

void client_thread::handle_query(std::shared_ptr<client_queries::disconnect> query) {
  m_running = false;
  query->promise.set_value();
}
```

Finally, the main thread can now interract with the client:

```cpp
m_client->query_set_ready().get();
```

---

## Bonus point: a “state machine” in the client

Some queries only make sense when the client is in a specific state, and messages from the server will determine that state

To make it easier, I decided to use an `std::variant` as well, allowing us to use `std::visit` or `std::holds_alternative` to verify that we are in the correct state:

```cpp
namespace game::client {
  using request_type = nlohmann::json;

  namespace client_states {
    struct connecting {};

    struct in_lobby {
      // ...
    };

    struct in_game {
      // ...
    };

    struct terminating {};
  }

  using client_state = std::variant<
    client_states::connecting,
    client_states::in_lobby,
    client_states::in_game,
    client_states::terminating
  >;

  class client_thread {
    private:
      // ...

      client_state m_state{client_states::connecting{}};

      // ...

    private:
      // ...

      void handle_state(
        client_states::connecting &state,
        const request_type &request
      );
      void handle_state(
        client_states::in_lobby &state,
        const request_type &request
      );
      void handle_state(
        client_states::in_game &state,
        const request_type &request
      );
      void handle_state(
        client_states::terminating &state,
        const request_type &request
      );
  };
}
```

We need to modify our thread loop a bit to accomodate for the `terminating` state:

```cpp
void client_thread::run(const std::string &host, int port) {
  // create socket

  try {
    m_running = true;

    while (m_running) {
      // wait and receive data from the socket
      // the ready promise is set once the server gave us our player_id

      handle_queries();
    }

    // notify the server we are disconnecting

    m_stop_promise.set_value();
  }
  catch (...) {
    m_stop_promise.set_exception(std::current_exception());
  }

  // empty the message queue to avoid blocking the main thread
  m_state = client_states::terminating{};
  handle_queries();

  // free socket
}
```

When receiving a request from the server, it will be dispatched to the correct state handler like this:

```cpp
std::visit([&](auto &&state) { handle_state(state, request); }, m_state);
```

Now, in a query handler, we can check that we are in the correct state before taking any action:

```cpp
void client_thread::handle_query(std::shared_ptr<client_queries::kick_player> query) {
  if (std::holds_alternative<client_states::in_lobby>(m_state)) {
    // send message to server
  }

  query->promise.set_value();
}
```

---

## Conclusion

In retrospective, this design have a lot of boilerplate code. I could probably make a parent class for both `server_thread` and `client_thread` to avoid the copy/pasta. Adding a new query imply:

- creating the structure and adding it to the variant
- creating the corresponding `handle_query()` method
- creating the corresponding `query_*()` method

I could probably create an interface defining the methods `publish` and `handle` that would be implemented by the query structures directly, thus avoiding the `std::variant` dispatch, but it works very well as-is, and it’s easy to extend/refactor. So I’ll let future me deal with those details 🙂

---

Feel free to clap for this article to give me more visibility 🙂

You can also join me on Discord:

{{< cta-link
  url="https://discord.gg/2RqWF8gkxU"
  title="Join the indielink Discord Server!"
  description="Check out the indielink community on Discord - hang out with 3 other members and enjoy free voice and text chat."
>}}

A Steam page and [itch.io](https://itch.io/) page for the game is currently in progress, stay tuned!

If you want to read the other devlogs, it’s here → [Devlogs Reading List](/reading-lists/warmonger-dynasty)
