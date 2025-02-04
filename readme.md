### **Explanation of the Code**

This code implements an AI-based airline assistant for a fictional airline called "TopAirline" using Python. It integrates the OpenAI API for chat interactions, image generation, and text-to-speech capabilities, and provides a Gradio UI for user interaction

* * *

<a href="https://www.loom.com/share/5469a64715684a80a289ea11015a6238" target="_blank">
</a>

<a href="https://www.loom.com/share/5469a64715684a80a289ea11015a6238" target="_blank">
    <img src="https://cdn.loom.com/sessions/thumbnails/5469a64715684a80a289ea11015a6238-2e7dac7dbb2236ce-full-play.gif" width="300">
</a>



### **1\. Library Imports**

The code imports various libraries, including:

-   `os`: For environment variable management.
-   `dotenv`: To load API keys from a `.env` file.
-   `json`: For handling JSON data.
-   `openai`: For accessing OpenAI's API.
-   `gradio`: To create a web-based UI.
-   `anthropic`: (Not used in the code, could be for Claude AI if integrated in the future.)
-   `base64`, `BytesIO`, `PIL (Image)`: For processing and displaying images.
-   `pydub`: For handling audio playback.
* * *

### **2\. API Key Loading**

```python
load_dotenv(override=True)
openai_key = os.getenv('OPENAI_API_KEY')
```

-   Loads the OpenAI API key from a `.env` file.
-   Ensures that any previously loaded values can be overridden.
* * *

### **3\. OpenAI Initialization**

```python
openai = OpenAI()
openai_model = 'gpt-4o-mini'
```

-   Instantiates an OpenAI client to interact with the API.
-   Specifies the model to use (`gpt-4o-mini`).
* * *

### **4\. System Message Definition**

```python
system_message = "You are an helpful AI assistant for an airline called topairline \
                Give short polite message, not more than 1 sentence \
                Always be accurate. If you don't know the answer, say so."
```

-   Defines the assistant's behavior, ensuring short, polite, and accurate responses.
* * *

### **5\. Ticket Prices Dictionary**

```python
ticket_prices = {
    "london": "$799",
    "paris": "$899",
    "tokyo": "$1400",
    ...
}
```

-   A dictionary mapping various cities to their respective ticket prices.
* * *

### **6\. `AirlineAssistant` Class**

The core functionality of the assistant is defined within this class.

#### **a) `__init__` Method (Constructor)**

```python
def __init__(self, destination_city):
    self.system_message = system_message
    self.ticket_prices = ticket_prices
    self.destination_city = destination_city
```

-   Initializes the assistant with the system message, ticket prices, and a specific destination city.
* * *

#### **b) `get_ticket_prices` Method**

```python
def get_ticket_prices(self):
    print(f"Tool get_ticket_prices for {self.destination_city}")
    city = self.destination_city.lower()
    return ticket_prices.get(city, "Unknown")
```

-   Retrieves the ticket price for the specified destination.
-   Returns "Unknown" if the city is not listed.
* * *

#### **c) `price_function` Method**

```python
def price_function(self):
    price_function = {
        'name': 'get_ticket_prices',
        'description': "Get the price of a return ticket to the destination city. ...",
        "parameters": {
            "type": "object",
            "properties": {
                "destination_city": {
                    "type": "string",
                    "description": "The city that the customer wants to travel to",
                },
            },
            "required": ["destination_city"],
            "additionalProperties": False
        }
    }
    return price_function
```

-   Defines a JSON schema for the function that retrieves ticket prices, which helps the AI assistant to understand how to handle user requests.
* * *

#### **d) `tool` Method**

```python
def tool(self):
    tools = [{'type': 'function', 'function': self.price_function()}]
    return tools
```

-   Returns a list of available tools (functions) that the assistant can use.
* * *

#### **e) `handle_tool_call` Method**

```python
def handle_tool_call(self, message):
    tool_call = message.tool_calls[0]
    arguments = json.loads(tool_call.function.arguments)
    city = arguments.get('destination_city')
    price = self.get_ticket_prices()
    response = {
        "role": "tool",
        "content": json.dumps({"destination_city": city, "price": price}),
        "tool_call_id": tool_call.id
    }
    return response, city
```

-   Handles AI tool calls by parsing the input request, retrieving the ticket price, and returning the formatted response.
* * *

#### **f) `artist` Method (Image Generation)**

```python
def artist(self):
    image_response = openai.images.generate(
        model='dall-e-3',
        prompt=f"An image representing a vacation in {self.destination_city}...",
        size="1024x1024",
        n=1,
        response_format="b64_json",
    )
    image_base64 = image_response.data[0].b64_json
    image_data = base64.b64decode(image_base64)
    return Image.open(BytesIO(image_data))
```

-   Generates a vacation-themed image of the destination city using OpenAI's DALL-E model.
-   Decodes and returns the image.
* * *

#### **g) `talker` Method (Text-to-Speech)**

```python
def talker(self, message):
    response = openai.audio.speech.create(
      model="tts-1",
      voice="onyx",
      input=message
    )
    audio_stream = BytesIO(response.content)
    audio = AudioSegment.from_file(audio_stream, format="mp3")
    play(audio)
```

-   Converts AI responses to audio using OpenAI's text-to-speech API.
-   Plays the audio using `pydub`.
* * *

#### **h) `chat` Method (Chat Handling)**

```python
def chat(self, history):
    messages = [{"role": "system", "content": system_message}] + history
    response = openai.chat.completions.create(model=openai_model, messages=messages, tools=tools)

    if response.choices[0].finish_reason == "tool_calls":
        message = response.choices[0].message
        response, city = handle_tool_call(message)
        image = artist(self.destination_city)
        response = openai.chat.completions.create(model=openai_model, messages=messages)

    reply = response.choices[0].message.content
    history += [{"role": "assistant", "content": reply}]
    talker(reply)

    return history, image
```

-   Handles the conversation flow, manages tool calls, and generates responses.
-   Adds images and audio as needed.
* * *

#### **i) `ui` Method (User Interface with Gradio)**

```python
def ui(self):
    with gr.Blocks() as ui:
        with gr.Row():
            chatbot = gr.Chatbot(height=500, type="messages")
            image_output = gr.Image(height=500)
        with gr.Row():
            entry = gr.Textbox(label="Chat with our AI Assistant:")
        with gr.Row():
            clear = gr.Button("Clear")

        def do_entry(message, history):
            history += [{"role": "user", "content": message}]
            return "", history

        entry.submit(do_entry, inputs=[entry, chatbot], outputs=[entry, chatbot]).then(
            chat, inputs=chatbot, outputs=[chatbot, image_output]
        )
        clear.click(lambda: None, inputs=None, outputs=chatbot, queue=False)

    ui.launch(inbrowser=True)
```

-   Creates a Gradio-based web interface with a chatbot, text input, and image display.
* * *

### **Summary of What the Code Does**

1.  Loads API credentials and sets up the assistant.
2.  Provides ticket prices based on city queries.
3.  Generates travel-related images using DALL-E.
4.  Uses text-to-speech for responses.
5.  Offers a Gradio-based UI for interaction.
* * *
