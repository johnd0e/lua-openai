# lua-openai

Bindings to the OpenAI HTTP API for Lua. Compatible with any socket library
that supports the LuaSocket request interface. Compatible with OpenResty using
[`lapis.nginx.http`](https://leafo.net/lapis/reference/utilities.html#making-http-requests).

## Install

Install using LuaRocks:

```bash
luarocks install lua-openai
```

## Quick Usage

```lua
local openai = require("openai")
local client = openai.new(os.getenv("OPENAI_API_KEY"))

local status, response = client:chat({
  {role = "system", content = "You are a Lua programmer"},
  {role = "user", content = "Write a 'Hello world' program in Lua"}
}, {
  model = "gpt-3.5-turbo", -- this is the default model
  temperature = 0.5
})

if status == 200 then
  -- the JSON response is automatically parsed into a Lua object
  print(response.choices[1].message.content)
end
```

## Chat Session Example

A chat session instance can be created to simplify managing the state of a back
and forth conversation with the ChatGPT API. Note that chat state is stored
locally in memory, each new message is appended to the list of messages, and
the output is automatically appended to the list for the next request. 

```lua
local openai = require("openai")
local client = openai.new(os.getenv("OPENAI_API_KEY"))

local chat = client:new_chat_session({
  -- provide an initial set of messages
  messages = {
    {role = "system", content = "You are an artist who likes colors"}
  }
})


-- returns the string response
print(chat:send("List your top 5 favorite colors"))

-- the chat history is sent on subsequent requests to continue the conversation
print(chat:send("Excluding the colors you just listed, tell me your favorite color"))
```

## Streaming Response Example

Under normal circumstances the API will wait until the entire response is
available before returning the response. Depending on the prompt this may take
some time. The streaming API can be used to read the output one chunk at a
time, allowing you to display content in real time as it is generated.

```lua
local openai = require("openai")
local client = openai.new(os.getenv("OPENAI_API_KEY"))

client:chat({
  {role = "system", content = "You work for Streak.Club, a website to track daily creative habits"},
  {role = "user", content = "Who do you work for?"}
}, {
  stream = true
}, function(chunk)
  io.stdout:write(chunk.content)
  io.stdout:flush()
end)

print() -- print a newline
```

## Documentation

The `openai` module returns a table with the following fields:

- `OpenAI`: A client for sending requests to the OpenAI API.
- `new`: An alias to `OpenAI` to create a new instance of the OpenAI client
- `ChatSession`: A class for managing chat sessions and history with the OpenAI API.
- `VERSION = "1.0.0"`: The current version of the library

### Classes

#### OpenAI

This class initializes a new OpenAI API client.

##### `new(api_key, config)`

Constructor for the OpenAI client.

- `api_key`: Your OpenAI API key.
- `config`: An optional table of configuration options, with the following shape:
  - `http_provider`: A string specifying the HTTP module name used for requests, or `nil`. If not provided, the library will automatically use "lapis.nginx.http" in an ngx environment, or "ssl.https" otherwise.

```lua
local openai = require("openai")
local api_key = "your-api-key"
local client = openai.new(api_key)
```

##### `client:new_chat_session(...)`

Creates a new ChatSession instance.

##### `client:chat(messages, opts, chunk_callback)`

Sends a request to the `/chat/completions` endpoint.

- `messages`: An array of message objects.
- `opts`: Additional options for the chat, such as model and temperature.
- `chunk_callback`: A function to be called for parsed streaming output when `stream = true` is passed to `opts`.

##### `client:completion(prompt, opts)`

Sends a request to the `/completions` endpoint.

- `prompt`: The prompt for the completion.
- `opts`: Additional options for the completion.

#### ChatSession

This class manages chat sessions and history with the OpenAI API. Typically created with `new_chat_session`

##### `new(client, opts)`

Constructor for the ChatSession.

- `client`: An instance of the OpenAI client.
- `opts`: An optional table of options.

##### `chat:append_message(m, ...)`

Appends a message to the chat history.

- `m`: A message object.

##### `chat:last_message()`

Returns the last message in the chat history.

##### `chat:send(message, stream_callback=nil)`

Appends a message to the chat history and triggers a completion with
`generate_response` and returns the response as a string. On failure, returns
`nil`, an error message, and the raw request response.

- `message`: A message object or a string.
- `stream_callback`: (optional) A function to enable streaming output.

By providing a `stream_callback`, the request will runin streaming mode. This function receives chunks as they are parsed from the response.

These chunks have the following format:

- `content`: A string containing the text of the assistant's generated response.

For example, a chunk might look like this:

```lua
{
  content = "This is a part of the assistant's response.",
}
```

##### `chat:generate_response(append_response, stream_callback=nil)`

Calls the OpenAI API to generate the next response for the stored chat history.
Returns the response as a string. On failure, returns `nil`, an error message,
and the raw request response.

- `append_response`: Whether the response should be appended to the chat history (default: true).
- `stream_callback`: (optional) A function to enable streaming output.

See `chat:send` for details on the `stream_callback`
