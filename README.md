# Caddy Authplugger

This plugin allows the user to provide generic authentication based on HTTP methods. It's main use case is to add (SSO) authentication to unsupported web services.

__Important Note: Authplugger has not yet reached a stable release, bugs may come and eat you.__ I already use it in production, so at least part of it seems to work very well - other parts may be incomplete ;)

## Building

As authplugger is not (yet) an official plugin of caddy, so you will have to build it yourself.

You can either put together your own `main.go` file, or use `simple.go` in the `test` directory.  
To build a caddy server just with authplugger as a plugin (and no others), go into the `test` directory and execute `go build simple.go`, the resulting file `simple` is your binary.

As there is currently (at the time of development) no way to log via Caddy, Authplugger will just print it's log lines.

## Usage

Authplugger works by filtering HTTP methods and request paths. The most common methods are included in the shortcuts _readonly_ `ro`, _read/write_ `rw`, _websockets_ `ws` and _any_ for all of the most common:

- `ro`: GET, HEAD, PROPFIND, OPTIONS, LOCK, UNLOCK
- `rw`: GET, HEAD, PROPFIND, OPTIONS, LOCK, UNLOCK, POST, PUT, DELETE, MKCOL, PROPPATCH
- `ws`: WEBSOCKET
- `any`: GET, HEAD, PROPFIND, OPTIONS, LOCK, UNLOCK, POST, PUT, DELETE, MKCOL, PROPPATCH, WEBSOCKET

If you want to specify your methods yourseld, be sure to not have any spaces between them: `GET,HEAD,...`.

If you prepend the list of methods with `~`, you can _invert_ their meaning, effectively turning the whitelist into a blacklist.

Authplugger works on a prefix basis. Every path provided to Authplugger matches the exact path and every path that starts with the provided path.

__Important Note: Authplugger is only secure if you can verify if the application you want to protect is compatible with Authplugger,__ meaning that it must conform to these standard HTTP methods to interact with the web service. Also, Authplugger can only deny websocket connections, but __cannot__ filter within them. If using Authplugger, you should always treat websocket as a full write access action.

##### Special Handling

The HTTP Methods `MOVE`, `COPY` and `PATCH` are handled a bit more special:

- `MOVE`: source path is treated as `DELETE` and destination path as `PUT`
- `COPY`: source path is treated as `GET` and destination path as `PUT`
- `PATCH`:
  - If a the `Destination` Header is present:
    - If `Action` Header is `copy`: source path is treated as `GET` and destination path as `PUT`
    - If `Action` Header is not `copy`: source path is treated as `DELETE` and destination path as `PUT`
  - If no `Destination` Header is present: treat as `PATCH`

## Backends

The different backends may be combined - They are handled in order of declaration in the Caddyfile.

Check out the test directory and play around with the different backends to get a feel for it.

Authplugger currently supports 3 different backends:

### API Auth

    authplugger api {
      name MyWebsite # name of website
      user http://localhost:8080/caddyapi # main authentication api
      permit http://localhost:8080/caddyapi/{{username}} # refetch a permit of a user
      login http://localhost:8080/login?next={{resource}} # redirect here for logging in (resource is original URL)
      add_prefix /api/resource /files # add prefices to returned paths
      add_no_prefix # if add_prefix is used, but you still want to also add the original paths
      cache 600 # how to long to cache authenticated users
      cleanup 3600 # when to clean out authenticated users
    }

__`user` Endpoint:__

Authplugger creates a request to the configure URL with:

- The original `Host` Header.
- The originating IP in the `X-Real-IP` Header.
- The originating IP in the `X-Forwarded-For` Header.
- The original protocol (`http` or `https`) in the `X-Forwarded-For` Header.
- The original BasicAuth credentials, if present.
- All cookies.

It expects a JSON Object in return with the following fields:

    {
      "BasicAuth":   false,
      "Cookie":      "cookieName=cookieValue",
      "Username":    "username",
      "Permissions": {}
    }

`BasicAuth` or `Cookie` are used to tell Authplugger how to identify this user in the future. Either set `BasicAuth` to true, or set `Cookie` to the authentication cookie.

Example:

    {
      "BasicAuth":   false,
      "Cookie":      "PHPSESSID=12345",
      "Username":    "tom",
      "Permissions": {
        "/tmp/": "rw",
        "/static": "ro",
        "/other": "GET,HEAD"
      }
    }

__`permit` Endpoint:__

Works very similar to the `user` endpoint, but instead of forwarding all these headers and cookies, the username is replaced in the URL.

__`login` Endpoint:__

If current permissions are insufficient to complete a request and the user is not yet authenticated, he is redirected to this URL.


### HTTP Basic Auth

    authplugger basic {
      user greg qwerty1 # This is greg, his password is qwerty1
      rw /tmp/ # he may read and write to /tmp/!

      default # applies to all logged-in users
      rw /api/users/0 #

      public # applies to everyone, also anonymous users
      ro /static # everyone may read stuff in the static folder
      ro /sw.js
      ro /api/auth/renew
      rw /api/users/0
      GET,HEAD /other
    }

### TLS Auth

This plugin requires TLS client authentication. It simply sets the CN to the username. You can use the `HTTP Basic Auth` or `API Auth` plugin for handling permissions.

    authplugger tls {
      setbasicauth admin admin # set basic auth on forwarded request (ie use tls client certs as a front for a simple password based service)
      setcookie token secret # set cookie on forwarded request
      setcookie language en
    }

## Combining Backends

When combining different backends, the backend defined earlier is always asked first. This handled as follows:

- Try to authenticate the user with every backend, stop if successful.
- If authenticated:
  - Check the user's permissions and default permissions for every backend, stop if allowed.
- Check the public permit for every backend, stop if allowed.
- Let the first backend to support login handle login.

## Other Options

There are also two other options regardless of backend:

    authplugger realm AuthPlugger # sets name
    authplugger allow_reading_parent_paths # applies read rights to parent paths
