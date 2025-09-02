# **Chatbot API Specification for Frontend Development**

**Version:** 2.0 (Component-Driven with Dedicated Errors)

**Status:** **Active & Final**

**Contact:** Backend Team Lead

## 1. Overview & Architecture

This document outlines the API for the B&S Virtual Assistant. The backend has been fully refactored to a modern, **component-driven JSON API**.

The core architectural principle is the **decoupling of backend logic from frontend presentation**. The backend no longer sends presentational HTML. Instead, it sends a structured description of the UI components that the frontend should render. The frontend (e.g., a React application) is solely responsible for interpreting this structure and rendering the appropriate visual components.

All communication is session-based. The frontend **must** correctly handle and send cookies (e.g., using `credentials: 'include'` in a `fetch` request) for the session to be maintained.

### 1.1. Additional Resources

-   **Live Demo Website:** [`https://lxchatbot-ui.azurewebsites.net`](https://lxchatbot-ui.azurewebsites.net/)

---

## 2. API Endpoints

### 2.1. Base URL

All endpoint paths are relative to this base URL: [`https://lxchatbot.azurewebsites.net/api/`](https://lxchatbot.azurewebsites.net/api/)

### 2.2. Main Chat Endpoint

This is the primary endpoint for all conversational interactions.

-   **Path:** `/chatbot`
-   **Authentication:** Session-based (requires cookies).

#### **A. Starting a New Conversation (GET Request)**

Used to begin a new chat session and receive the initial welcome message.

-   **Method:** `GET`
-   **Response (Success - 200 OK):**
    -   **Content-Type:** `application/json`
    -   **Body:** A JSON object containing the initial bot message components and a unique session ID. The `chatbot_session` value **must** be stored by the frontend and sent back in the body of all subsequent `POST` requests.
    -   **Example Response:**
        ```json
        {
            "BOT": {
                "components": [
                    {
                        "type": "text",
                        "payload": {
                            "text": "Hi there! Welcome to BNS group.<br>I'm your virtual assistant.<br>Are you an existing customer?"
                        }
                    },
                    {
                        "type": "button_options",
                        "payload": {
                            "options": [
                                { "label": "Yes, I am", "value": "yes" },
                                { "label": "No, I'm new", "value": "no" }
                            ]
                        }
                    }
                ]
            },
            "chatbot_session": "a_unique_session_id_string",
            "is_terminal": false
        }
        ```

#### **B. Continuing a Conversation (POST Request)**

Used to send a user's message or action to the backend and receive the bot's reply.

-   **Method:** `POST`
-   **Headers:** `Content-Type: application/json`
-   **Request Body:** A JSON object containing the user's message/action and the session ID.
    ```json
    {
        "message": "user_input_or_button_value",
        "chatbot_session": "the_session_id_from_the_GET_request"
    }
    ```
-   **Response (Success - 200 OK):**
    -   **Content-Type:** `application/json`
    -   **Body:** A JSON object containing the bot's response components. The `is_terminal` key will be `true` if the conversation has reached a definitive end.
    -   **Example Response:**
        ```json
        {
            "BOT": {
                "components": [
                    // ... one or more component objects ...
                ]
            },
            "is_terminal": false
        }
        ```

### 2.3. Image Upload Endpoint

For uploading an image specifically in the "Damaged Product" flow.

-   **Path:** `/upload-image/`
-   **Method:** `POST`
-   **Content-Type:** `multipart/form-data`
-   **Request Body:** The request must be form data containing the image file under the key `image`.
-   **Response (Success - 200 OK):**
    ```json
    {
        "message": "Image uploaded successfully!<br><br>Please enter the Damaged Quantity in the chat:",
        "uploaded_image_url": "https://url.to/the/uploaded/image.png"
    }
    ```

### 2.4. Session Close Endpoint

To inform the backend that the user has closed the chat window. This is an optional "fire-and-forget" call.

-   **Path:** `/session-close/`
-   **Method:** `POST`
-   **Request Body:**
    ```json
    {
        "chatbot_session": "the_current_session_id"
    }
    ```
-   **Response (Success - 200 OK):**
    ```json
    {
        "message": "Session scheduled for closure."
    }
    ```

---

## 3. UI Component Reference

The frontend's primary responsibility is to render UI based on the `type` key inside each object of the `BOT.components` array.

### Component Type: `text`

The most basic component. Used for displaying all standard text-based messages from the bot.

-   **`type`:** `"text"`
-   **`payload`:**
    -   `text` (string): The message content. **This string may contain HTML tags** (`<br>`, `<b>`) and must be rendered accordingly (e.g., using `dangerouslySetInnerHTML` in React).
-   **Example Payload:**
    ```json
    {
        "type": "text",
        "payload": {
            "text": "Please provide your <b>account number</b> to proceed further."
        }
    }
    ```

### Component Type: `button_options`

A group of interactive buttons that present choices to the user.

-   **`type`:** `"button_options"`
-   **`payload`:**
    -   `options` (array of objects): A list of buttons to display. Each object has:
        -   `label` (string): The text displayed on the button.
        -   `value` (string): The value that must be sent back to the backend in the `message` field when this button is clicked.
-   **Example Payload:**
    ```json
    {
        "type": "button_options",
        "payload": {
            "options": [
                { "label": "Delivery Issue", "value": "1" },
                { "label": "Product Query", "value": "2" }
            ]
        }
    }
    ```

### Component Type: `product_carousel`

A horizontally scrolling container of product cards. Used to display a list of products.

-   **`type`:** `"product_carousel"`
-   **`payload`:**
    -   `products` (array of objects): A list of products. Each object has:
        -   `index` (string): The unique identifier for the product (e.g., "1"). When a user selects this product, this `index` value must be sent back to the backend.
        -   `name` (string): The full name of the product.
        -   `quantity` (string): The quantity of the product.
        -   `price` (string): The unit price of the product.
-   **Example Payload:**
    ```json
    {
        "type": "product_carousel",
        "payload": {
            "products": [
                {
                    "index": "1",
                    "name": "Amoxicillin-Clavulanate 625mg Tablets",
                    "quantity": "10.0",
                    "price": "6.25"
                }
            ]
        }
    }
    ```

### Component Type: `selected_product_card`

A static display card showing the details of a product the user has just selected.

-   **`type`:** `"selected_product_card"`
-   **`payload`:**
    -   `product` (object): An object containing the details of the selected product.
        -   `name`, `quantity`, `price`, `total_price` (all strings)
-   **Example Payload:**
    ```json
    {
        "type": "selected_product_card",
        "payload": {
            "product": {
                "name": "Amoxicillin-Clavulanate 625mg Tablets",
                "quantity": "10.0",
                "price": "6.25",
                "total_price": "62.5"
            }
        }
    }
    ```

### Component Type: `error`

**New in v2.1.** A dedicated component for displaying user-facing validation or error messages. This should be rendered with a distinct, attention-grabbing style.

-   **`type`:** `"error"`
-   **`payload`:**
    -   `text` (string): The error message content. This is plain text, not HTML.
-   **Example Payload:**
    ```json
    {
      "type": "error",
      "payload": { "text": "Invalid OTP entered. You have 2 attempt(s) left." }
    }
    ```

---

## 4. Error Handling

### 4.1 User-Facing Errors

If the user provides invalid input, the backend will now prepend an `error` component to the `components` array in the response.

-   **Mechanism:** An object with `type: "error"` will be the first item in the `components` list. The original prompt components (e.g., `text`, `button_options`) will follow it.
-   **Frontend Action:** The frontend must render this `error` component with a distinct visual style to inform the user of their mistake before re-displaying the prompt.
-   **Example Error Response:**
    ```json
    {
        "BOT": {
            "components": [
                {
                    "type": "error",
                    "payload": { "text": "Invalid input. Please select an option." }
                },
                {
                    "type": "text",
                    "payload": { "text": "Are you an existing customer?" }
                },
                {
                    "type": "button_options",
                    "payload": { "options": [ /* ... */ ] }
                }
            ]
        },
        "is_terminal": false
    }
    ```

### 4.2 Server Errors

In case of an unexpected server-side exception:

-   **Status Code:** `500 Internal Server Error`
-   **Response Body:** A simplified response containing a single `error` component.
    ```json
    {
        "BOT": {
            "components": [
                {
                    "type": "error",
                    "payload": {
                        "text": "I'm sorry, an unexpected error occurred. Please try again."
                    }
                }
            ]
        },
        "is_terminal": true
    }
    ```

## 5. Document History

-   **v2.1 (Current):** Introduced a dedicated `error` component type for cleaner, more semantic error handling. Refined API contract.
-   **v2.0:** Complete transition to a component-driven JSON API. Deprecated all HTML-in-JSON responses. Introduced `text`, `button_options`, `product_carousel`, and `selected_product_card` components.
-   **v1.0 (Legacy):** Mixed JSON and HTML responses. Responses contained a single `message` key with pre-rendered HTML. Deprecated.
  
