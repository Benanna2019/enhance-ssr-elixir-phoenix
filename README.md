# SsrWebComponentsOnTheBeam

Setup Steps

1. Install Rust (if not already installed) - run this in a terminal `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
2. Install elixir - probably just use homebrew
3. Make sure you have postgres setup and installed - (I use postgresapp.com)

Create a New Phoenix Project (or clone this project)

- run `mix phx.new name_of_app --live`
- You have to give the app a name or it will fail. The `--live` option is to say this is a LiveView project

Add extism as a dependency

- in the mix.exs file add `{:extism, "1.0.0"}`
- the run `mix deps.get`

Adding enhance-ssr/wasm

- Create wasm directory
- Download the enhance wasm file into local directory - `curl -L [https://github.com/enhance-dev/enhance-ssr-wasm/releases/download/v0.0.3/enhance-ssr.wasm.gz](https://github.com/enhance-dev/enhance-ssr-wasm/releases/download/v0.0.3/enhance-ssr.wasm.gz) | gunzip > wasm/enhance-ssr.wasm`

Creating an Extism Plugin

- Look at extism elixir docs → Show that we need to create a plugin in a very specific way
  [https://extism.org/docs/quickstart/host-quickstart/](https://extism.org/docs/quickstart/host-quickstart/)
- Create an Elixir/Phoenix module that ‘creates_plugin’

```elixir
defmodule SsrWebComponentsOnTheBeam.ConvertComponents do
    @wasm_plugin_path Path.expand("../../../wasm/enhance-ssr.wasm", __DIR__)

    def create_plugin do
        # Define the path to your local WASM file

        IO.inspect "Creating plugin with path: #{@wasm_plugin_path}"

        # Create the manifest with the local file path
        manifest = %{wasm: [%{path: @wasm_plugin_path}]}

        # Create the plugin with Extism.Plugin.new
        case Extism.Plugin.new(manifest, true) do
          {:ok, plugin} ->
            {:ok, plugin}

          {:error, reason} ->
            {:error, reason}
        end
      end
end
```

- Pull up enhance documentation for what enhance expects as a function signature
  [GitHub - enhance-dev/enhance-ssr-wasm: Enhance SSR compiled for WASM](https://github.com/enhance-dev/enhance-ssr-wasm?tab=readme-ov-file#usage)
- Create a ‘call_enhance_plugin’ function
  ```elixir
  defmodule SsrWebComponentsOnTheBeam.ConvertComponents do
      @wasm_plugin_path Path.expand("../../../wasm/enhance-ssr.wasm", __DIR__)

      def create_plugin do
          # Define the path to your local WASM file

          IO.inspect "Creating plugin with path: #{@wasm_plugin_path}"

          # Create the manifest with the local file path
          manifest = %{wasm: [%{path: @wasm_plugin_path}]}

          # Create the plugin with Extism.Plugin.new
          case Extism.Plugin.new(manifest, true) do
            {:ok, plugin} ->
              {:ok, plugin}

            {:error, reason} ->
              {:error, reason}
          end
        end

      def call_enhance_plugin(plugin, data) do
        Extism.Plugin.call(plugin, "ssr", Jason.encode!(data))
      end
  end
  ```
- decode the output which should just be a variable called enhance
- get the document off of the enhance output and return in in a the raw function in a `<%= =>` expression in a `~H` template

```elixir
defmodule SsrWebComponentsOnTheBeam.EnhanceLive do
  use SsrWebComponentsOnTheBeam, :live_view
  use Phoenix.Component

  alias SsrWebComponentsOnTheBeam.ConvertComponents

  def mount(_params, _session, socket) do
    socket =
      socket
      |> assign(:color, "text-red-500")

    {:ok, socket}
  end

  def render(assigns) do
    ~H"""
    <.enhance_header id='my-header' color={@color} />

    <button phx-click="change-color">Change color to red</button>
    """
  end

def enhance_header(assigns) do

    IO.puts "assigns: #{inspect(assigns)}"

    data = %{
        markup: "<my-header id='my-header' color=#{assigns.color}>Hello World</my-header>",
        elements: %{
          "my-header":
            "function MyHeader({ html, state }) {
              const { attrs, store } = state
              const attrs_color = attrs['color']
              const id = attrs['id']
              const store_works = store['readFromStore']
              return html`<h1 class='${attrs_color}'><slot></slot></h1><p>store works: ${store_works} </p><p>attrs id: ${id} </p><p>attrs color: ${attrs_color} </p>`
            }",
        },
        initialState: %{ readFromStore: "true" },
      }

    {:ok, plugin} = ConvertComponents.create_plugin()

    {:ok, output} = ConvertComponents.call_enhance_plugin(plugin, data)

    html = Jason.decode!(output)

    ~H"""
      <div>
        <%= raw(html["document"]) %>
      </div>
    """

  end

  def handle_event("change-color", _, socket) do
    {:noreply, assign(socket, :color, "text-blue-500")}
  end

 end
```
