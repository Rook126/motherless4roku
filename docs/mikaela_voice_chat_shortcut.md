# Mikaela Voice Chat Shortcut

This iOS Shortcut provides a looping, voice-driven chat experience using the OpenAI `responses` API. The shortcut asks for spoken input, sends it to a lightweight model, and speaks the reply back at a slowed speech rate for clarity.

## What the shortcut does
- Prompts you with **"What do you want to say to Mikaela?"**.
- Sends the response payload to `https://api.openai.com/v1/responses` with the `gpt-4o-mini` model.
- Includes stylistic instructions so replies stay **confident, bossy, bubbly, and flirty but non-explicit**.
- Reads the assistant reply aloud (`en-US`, rate `0.5`) and then repeats the prompt so the conversation can continue hands-free.

## Setup steps
1. Open **Shortcuts** on iOS and create a new shortcut.
2. Add the following actions in order:
   - **Ask for Input** with the prompt `What do you want to say to Mikaela?`.
   - **Get Contents of URL** configured as:
     - Method: `POST`
     - URL: `https://api.openai.com/v1/responses`
     - Headers: `Authorization: Bearer YOUR_OPENAI_API_KEY` and `Content-Type: application/json`
     - Request Body (JSON):
       ```json
       {
         "model": "gpt-4o-mini",
         "instructions": "You are Mikaela: confident, bossy, bubbly, dramatic queen‑bee cyborg. Keep it playful and flirty but non‑explicit, respectful, and consent‑focused. No sexual content or explicit descriptions. Keep replies short, witty, and spoken‑friendly.",
         "input": [
           {
             "role": "user",
             "content": [
               {
                 "type": "input_text",
                 "text": "Provided by Shortcut input"
               }
             ]
           }
         ]
       }
       ```
   - **Get Dictionary Value** from the API response to extract the text. The response structure is `output_items[0].text`, so:
     - First get `output_items` from the response dictionary
     - Then get the first item (index 0)
     - Finally get the `text` value from that item
   - **Speak Text** with language `en-US` and rate `0.5`.
   - **Repeat** set to a high count (e.g., `9999`) to keep the conversation going.
3. Replace `YOUR_OPENAI_API_KEY` with a valid API key before running the shortcut.

## API Response Format
The OpenAI `/v1/responses` endpoint returns a JSON object with this structure:
```json
{
  "id": "response-xxxx",
  "object": "response",
  "created": 1234567890,
  "model": "gpt-4o-mini",
  "output_items": [
    {
      "type": "output_text",
      "text": "Mikaela's response here..."
    }
  ],
  "usage": {
    "prompt_tokens": 50,
    "completion_tokens": 30,
    "total_tokens": 80
  }
}
```

## Usage tips
- Start the shortcut and speak naturally; each reply is read back before prompting you again.
- Adjust the speech rate or model if you prefer faster narration or richer responses.
- To stop the loop, end the shortcut from the Shortcuts app or via the stop control in the running widget.

## Troubleshooting
- If the shortcut fails to speak the response, verify that you're correctly extracting the text from `output_items[0].text` in the API response.
- Ensure your OpenAI API key has access to the `gpt-4o-mini` model and the `/v1/responses` endpoint.
- If you get authentication errors, double-check that the Authorization header is formatted as `Bearer YOUR_API_KEY` with the correct key.
