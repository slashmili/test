
# MyApp

base branch
---------------
```
mix new my_app --umbrella
cd my_app/apps/
mix phoenix.new api --no-html --no-brunch
```
base-api branch
---------------
```
cd api
mix phoenix.gen.json Vault vaults content:string
<follow instruction>
mix ecto.create
mix ecto.migrate
mix phoenix.server
```
POSTMAN -> GET -> POST
->
```
{
  "data": {
    "id": 1,
    "secret": "fbabb6fb-f359-446c-9bed-99614a55",
	"content": "UqPXRGoxYqvpFHeG5+EBxVRWcOV/ftEWSHdslXZqIv0="
  }
}
```
->

==encrypt-data branch==
---------------

```
cd ../
mix new encryption --sup
```

check it in console

==encrypt-data-in-controller branch==
---------------

{ :uuid, "~> 1.1" }

{:encryption, in_umbrella: true},

change controller to encrypte the data and returns secret and ecnrypted data

```diff
diff --git a/apps/api/mix.exs b/apps/api/mix.exs
index d1fcf53..bdb2381 100644
--- a/apps/api/mix.exs
+++ b/apps/api/mix.exs
@@ -40,6 +40,7 @@ defmodule Api.Mixfile do
      {:postgrex, ">= 0.0.0"},
      {:gettext, "~> 0.11"},
      { :uuid, "~> 1.1" },
+     {:encryption, in_umbrella: true},
      {:cowboy, "~> 1.0"}]
   end

diff --git a/apps/api/web/controllers/vault_controller.ex b/apps/api/web/controllers/vault_controller.ex
index ce4e29a..20d3b2f 100644
--- a/apps/api/web/controllers/vault_controller.ex
+++ b/apps/api/web/controllers/vault_controller.ex
@@ -8,8 +8,10 @@ defmodule Api.VaultController do
     render(conn, "index.json", vaults: vaults)
   end

-  def create(conn, %{"vault" => vault_params}) do
-    changeset = Vault.changeset(%Vault{}, vault_params)
+  def create(conn, %{"vault" => %{"content" => content}}) do
+    secret = UUID.uuid4
+    {:ok, content} = Encryption.encrypt(content, secret)
+    changeset = Vault.changeset(%Vault{}, %{content: content, secret: secret})

     case Repo.insert(changeset) do
       {:ok, vault} ->
diff --git a/apps/api/web/models/vault.ex b/apps/api/web/models/vault.ex
index b1663b6..1cd0c29 100644
--- a/apps/api/web/models/vault.ex
+++ b/apps/api/web/models/vault.ex
@@ -3,6 +3,7 @@ defmodule Api.Vault do

   schema "vaults" do
     field :content, :string
+    field :secret, :string, virtual: true

     timestamps()
   end
@@ -12,7 +13,7 @@ defmodule Api.Vault do
   """
   def changeset(struct, params \\ %{}) do
     struct
-    |> cast(params, [:content])
+    |> cast(params, [:content, :secret])
     |> validate_required([:content])
   end
 end
diff --git a/apps/api/web/views/vault_view.ex b/apps/api/web/views/vault_view.ex
index 3c6e081..5e11ddc 100644
--- a/apps/api/web/views/vault_view.ex
+++ b/apps/api/web/views/vault_view.ex
@@ -11,6 +11,7 @@ defmodule Api.VaultView do

   def render("vault.json", %{vault: vault}) do
     %{id: vault.id,
-      content: vault.content}
+      content: vault.content,
+      secret: vault.secret}
   end
 end
```

==decrypt-data-in-controller branch==
---------------

```diff
-  def show(conn, %{"id" => id}) do
+  def show(conn, %{"id" => id, "secret" => secret}) do
     vault = Repo.get!(Vault, id)
-    render(conn, "show.json", vault: vault)
+    case Encryption.decrypt(vault.content, secret) do
+      {:error, _} ->
+        conn
+        |> put_status(:unprocessable_entity)
+        |> render(Api.ChangesetView, "error.json", changeset: Vault.changeset(vault))
+      {:ok, content} ->
+        vault = %{vault | content: content, secret: secret}
+        render(conn, "show.json", vault: vault)
+    end
   end
```

==decrypt-data-in-controller branch==
---------------
```
iex --sname node01@localhost -S mix phoenix.server
cd apps/encryption/
iex --sname node02@localhost -S mix
Node.connect(:"node01@localhost")
Node.list
Node.spawn_link(:"node01@localhost", fn -> IO.inspect node end)
Node.spawn_link(:"node01@localhost", fn -> IO.inspect Encryption.encrypt("hello", "1123343434343434343434343434343343434") end)
```

==encrypte-gen-server branch===

```diff
diff --git a/apps/encryption/lib/encryption.ex b/apps/encryption/lib/encryption.ex
index 6a8b7e0..5cbc85f 100644
--- a/apps/encryption/lib/encryption.ex
+++ b/apps/encryption/lib/encryption.ex
@@ -10,6 +10,7 @@ defmodule Encryption do
     children = [
       # Starts a worker by calling: Encryption.Worker.start_link(arg1, arg2, arg3)
       # worker(Encryption.Worker, [arg1, arg2, arg3]),
+      worker(Encryption.Es256, []),
     ]

     # See http://elixir-lang.org/docs/stable/elixir/Supervisor.html
diff --git a/apps/encryption/lib/es_2564.ex b/apps/encryption/lib/es_2564.ex
index d13bb5a..d94819e 100644
--- a/apps/encryption/lib/es_2564.ex
+++ b/apps/encryption/lib/es_2564.ex
@@ -1,5 +1,11 @@
 defmodule Encryption.Es256 do

+  use GenServer
+
+  def start_link do
+    GenServer.start_link(__MODULE__, [], name: __MODULE__)
+  end
+
   def gen_iv do
     :crypto.strong_rand_bytes(16)
   end
@@ -20,8 +26,19 @@ defmodule Encryption.Es256 do
     encrypt(plain_text, binary_part(secret, 0, 32), iv)
   end
   def encrypt(plain_text, secret, iv) do
+    GenServer.call(__MODULE__, {:encrypt, plain_text, secret, iv})
+  end
+
+
+  def handle_call({:encrypt, plain_text, secret, iv}, _, state) do
     cipher_text = :crypto.block_encrypt(:aes_cbc256, secret, iv, pkcs7_pad(plain_text))
-    {:ok, Base.encode64(iv <> cipher_text)}
+    {:reply, {:ok, Base.encode64(iv <> cipher_text)}, state}
+  end
+
+  def handle_call({:decrypt, cipher_text, secret}, _, state) do
+    <<iv :: binary-size(16), cipher :: binary>> = Base.decode64!(cipher_text)
+    text_with_pad = :crypto.block_decrypt(:aes_cbc256, secret, iv, cipher)
+    {:reply, pkcs7_unpad(text_with_pad), state}
   end

   @doc """
@@ -37,9 +54,7 @@ defmodule Encryption.Es256 do
     decrypt(cipher_text, binary_part(secret, 0, 32))
   end
   def decrypt(cipher_text, secret) do
-    <<iv :: binary-size(16), cipher :: binary>> = Base.decode64!(cipher_text)
-    text_with_pad = :crypto.block_decrypt(:aes_cbc256, secret, iv, cipher)
-    pkcs7_unpad(text_with_pad)
+    GenServer.call(__MODULE__, {:decrypt, cipher_text, secret})
   end

   # Pads a message using the PKCS #7 cryptographic message syntax.
@@ -73,4 +88,3 @@ defmodule Encryption.Es256 do
     end
   end
 end
```

== PG

```
:pg2.create("encryption")
:pg2.join("encryption", self)
:pg2.get_members("encryption")
```

===distrubited-encryption branch==
---------------

```diff
diff --git a/apps/encryption/lib/encryption.ex b/apps/encryption/lib/encryption.ex
index 5cbc85f..da7d44b 100644
--- a/apps/encryption/lib/encryption.ex
+++ b/apps/encryption/lib/encryption.ex
@@ -20,10 +20,16 @@ defmodule Encryption do
   end

   def encrypt(data, secret) do
-    Encryption.Es256.encrypt(data, secret)
+    Encryption.Es256.encrypt(random_process, data, secret)
   end

   def decrypt(data, secret) do
-    Encryption.Es256.decrypt(data, secret)
+    Encryption.Es256.decrypt(random_process, data, secret)
+  end
+
+  defp random_process do
+    "encryption"
+    |> :pg2.get_members
+    |> Enum.random
   end
 end
diff --git a/apps/encryption/lib/es_2564.ex b/apps/encryption/lib/es_2564.ex
index d94819e..919f85e 100644
--- a/apps/encryption/lib/es_2564.ex
+++ b/apps/encryption/lib/es_2564.ex
@@ -2,10 +2,17 @@ defmodule Encryption.Es256 do

   use GenServer

+  require Logger
   def start_link do
     GenServer.start_link(__MODULE__, [], name: __MODULE__)
   end

+  def init(_) do
+    :pg2.create("encryption")
+    :pg2.join("encryption", self)
+    {:ok, %{}}
+  end
+
   def gen_iv do
     :crypto.strong_rand_bytes(16)
   end
@@ -16,26 +23,28 @@ defmodule Encryption.Es256 do
       iex> Encryption.Es256.encrypt("vault data", "fbabb6fb-f359-446c-9bed-99614a55")
       {:ok, "2B3Q+TXfX2QhTbzgbBZYbFILZdVCTR4JSVLZS+PHTes="}
   """
-  def encrypt(plain_text, secret) do
-    encrypt(plain_text, secret, gen_iv)
+  def encrypt(pid, plain_text, secret) do
+    encrypt(pid, plain_text, secret, gen_iv)
   end
-  def encrypt(_plain_text, secret, _iv) when bit_size(secret) < 256 do
+  def encrypt(_pid, _plain_text, secret, _iv) when bit_size(secret) < 256 do
     {:error, :invalid_secret}
   end
-  def encrypt(plain_text, secret, iv) when bit_size(secret) > 256 do
-    encrypt(plain_text, binary_part(secret, 0, 32), iv)
+  def encrypt(pid, plain_text, secret, iv) when bit_size(secret) > 256 do
+    encrypt(pid, plain_text, binary_part(secret, 0, 32), iv)
   end
-  def encrypt(plain_text, secret, iv) do
-    GenServer.call(__MODULE__, {:encrypt, plain_text, secret, iv})
+  def encrypt(pid, plain_text, secret, iv) do
+    GenServer.call(pid, {:encrypt, plain_text, secret, iv})
   end


   def handle_call({:encrypt, plain_text, secret, iv}, _, state) do
+    Logger.debug("Calculated in #{inspect node}")
     cipher_text = :crypto.block_encrypt(:aes_cbc256, secret, iv, pkcs7_pad(plain_text))
     {:reply, {:ok, Base.encode64(iv <> cipher_text)}, state}
   end

   def handle_call({:decrypt, cipher_text, secret}, _, state) do
+    Logger.debug("Calculated in #{inspect node}")
     <<iv :: binary-size(16), cipher :: binary>> = Base.decode64!(cipher_text)
     text_with_pad = :crypto.block_decrypt(:aes_cbc256, secret, iv, cipher)
     {:reply, pkcs7_unpad(text_with_pad), state}
@@ -47,14 +56,14 @@ defmodule Encryption.Es256 do
       iex> Encryption.Es256.decrypt("2B3Q+TXfX2QhTbzgbBZYbFILZdVCTR4JSVLZS+PHTes=", "fbabb6fb-f359-446c-9bed-99614a55")
       {:ok, "vault data"}
   """
-  def decrypt(_cipher_text, secret) when bit_size(secret) < 256 do
+  def decrypt(_pid, _cipher_text, secret) when bit_size(secret) < 256 do
     {:error, :invalid_secret}
   end
-  def decrypt(cipher_text, secret)  when bit_size(secret) > 256 do
-    decrypt(cipher_text, binary_part(secret, 0, 32))
+  def decrypt(pid, cipher_text, secret)  when bit_size(secret) > 256 do
+    decrypt(pid, cipher_text, binary_part(secret, 0, 32))
   end
-  def decrypt(cipher_text, secret) do
-    GenServer.call(__MODULE__, {:decrypt, cipher_text, secret})
+  def decrypt(pid, cipher_text, secret) do
+    GenServer.call(pid, {:decrypt, cipher_text, secret})
   end

   # Pads a message using the PKCS #7 cryptographic message syntax.
  ```

