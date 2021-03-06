# Cipher

[![Build Status](https://api.travis-ci.org/rubencaro/cipher.svg)](https://travis-ci.org/rubencaro/cipher)
[![Hex Version](http://img.shields.io/hexpm/v/cipher.svg?style=flat)](https://hex.pm/packages/cipher)
[![Hex Version](http://img.shields.io/hexpm/dt/cipher.svg?style=flat)](https://hex.pm/packages/cipher)

Elixir crypto library to encrypt/decrypt arbitrary binaries. It uses
[Erlang Crypto](http://www.erlang.org/doc/man/crypto.html), so it's not big
deal. Mostly a collection of helpers wrapping it.

This library allows us to use a crypted key to validate signed requests, with a
cipher compatible with
[this one](https://gist.github.com/rubencaro/9545060#file-gistfile3-ex).
This way it can be used from Python, Ruby or Elixir apps.

## Use

Just add `{:cipher, ">= 1.0.0"}` to your `mix.exs`.

Then add your keys to `config.exs`, **they are needed to compile `Cipher`**:
```elixir
config :cipher, keyphrase: "testiekeyphraseforcipher",
                ivphrase: "testieivphraseforcipher",
                magic_token: "magictoken"
```

Then you may use any of the given helpers:

## Encrypt/Decrypt binaries

Now you can use bare `encrypt/1` and `decrypt/1`:
```elixir
"secret"
|> Cipher.encrypt  # "KSHHdx0uyveYGY5PHqLAKw%3D%3D"
|> Cipher.decrypt  # "secret"
```

## Cipher/Parse JSON

`cipher/1` and `parse/1`. Just as `encrypt/1` and `decrypt/1` but for JSON.

```elixir
%{"hola": " qué tal ｸｿ"}
|> Cipher.cipher  # "qW0Voj3h4nglx4NPy8aLXVY5ze5V3OBu5IoaQTMUUbU%3D"
|> Cipher.parse  #  {:ok, %{"hola" => " qué tal ｸｿ"}}
```

## Sign/Validate a URL

Here you use `sign_url/2` and `validate_signed_url/1`.

`sign_url` will add a `signature` parameter to the end of the query string. It's a crypted hash based on the given path.

```elixir
"/bla/bla?p1=1&p2=2"
|> Cipher.sign_url  # "/bla/bla?p1=1&p2=2&signature=4B6WOiuD9N39K7p%2BnqNIljGh5F%2F%2BnHRQGZC9ih%2Bh%2BHGZc8Tz0KdRJXC%2B5M%2B8%2BHZ2mAXPh3jQcSRieTq4dGm5Ng%3D%3D"
```

`validate_signed_url` must be given an url with the `signature` parameter on the query string just as `sign_url` returned it. It will pop it, and validate that it corresponds with the rest of the URL.

```elixir
"/bla/bla?p1=1&p2=2&signature=4B6WOiuD9N39K7p%2BnqNIljGh5F%2F%2BnHRQGZC9ih%2Bh%2BHGZc8Tz0KdRJXC%2B5M%2B8%2BHZ2mAXPh3jQcSRieTq4dGm5Ng%3D%3D"
|> Cipher.validate_signed_url  # {:ok, %{"md5" => "86e359da7ab4886f3525ac2b9c5edc5b  613146"}}
```

Any changes to the signed URL `"/bla/bla?p1=1&p2=2"` will return `{:error, reason}` when validated.

```elixir
"/bla/bla?p1=1&p2=3&signature=4B6WOiuD9N39K7p%2BnqNIljGh5F%2F%2BnHRQGZC9ih%2Bh%2BHGZc8Tz0KdRJXC%2B5M%2B8%2BHZ2mAXPh3jQcSRieTq4dGm5Ng%3D%3D"
|> Cipher.validate_signed_url  # {:error, "Bad signature"}
```

### Ignored params

You can choose to sign a URL but then add some parameters to the query string that may not be signed, such as a `cachebuster`.

For that you can use `sign_url/2`, which accepts a payload to be included on the crypted signature. If you add a `ignore` list, then any parameter on that list will be accepted.

```elixir
signed = "/bla/bla?p1=1&p2=2" |> Cipher.sign_url(ignore: ["cachebuster"])

"#{signed}&cachebuster=123456789" |> Cipher.validate_signed_url  
#   {:ok,
#   %{"ignore" => ["cachebuster"],
#     "md5" => "86e359da7ab4886f3525ac2b9c5edc5b  971036"}}

"#{signed}&cachebuster=123456789&other=parm"
|> Cipher.validate_signed_url  # {:error, %MatchError{term: :error}}

```

### Concealed params

Note you can use `sign_url/2` to pass any data within the signature itself, just as you do with the `ignore` list. Any payload will be returned by `validate_signed_url/1`.

```elixir
signed = "/bla/bla?p1=1" |> Cipher.sign_url(mydata: "yes, any data")
signed |> Cipher.validate_signed_url
#  {:ok,
#   %{"md5" => "eacac4224aef3bfabee309ee2f95c1e8  176303",
#     "mydata" => "yes, any data"}}
```

If you want to pass cipher data on your URLs you could also use straight `cipher/1` and `parse/1`.

## Sign/Validate body

The same as signing a complete URL with query string, but for PUT/POST requests, where the signed data is in the body.

Helpers are `sign_url_from_body/2` and `validate_signed_body/1`. They put and validate the signature on the query string, so the body is untouched.

```elixir
url = "/bla/bla"
body = Poison.encode! %{"hola": " qué tal ｸｿ"}
signed = Cipher.sign_url_from_body(url, body, ignore: ["cb"])
# "/bla/bla?signature=HdlsREqEP9hJmP94..."
{:ok, _} = "#{signed}" |> Cipher.validate_signed_body(body)
{:ok, _} = "#{signed}&cb=123456" |> Cipher.validate_signed_body(body)
assert {:error, _} = "#{signed}&other=123456" |> Cipher.validate_signed_body(body)
```

## Magic Token

This is a master signature. If you put this binary as `signature` on your url, then it will always validate. This is useful for development, debugging, private network use, etc. You put your chosen `magic_token` on your `config.exs` and you are good to go.

```elixir
"/bla?any=thing&signature=mymagictoken"
|> Cipher.validate_signed_url  # {:ok, %{}}
```

## TODOs

* Improve error messages
