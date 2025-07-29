## amazon-lex-helper

This repository contains a comprehensive set of helper classes to handle Amazon Lex V2 responses and create custom requests with enhanced features.

### Core Functionality
* **LexEvent**: Amazon Lex event class with enhanced methods for sentiment analysis, confidence scores, and multi-valued slots
* **LexResponse**: Builder to create Amazon Lex responses with support for rich message types (SSML, Image Response Cards, Custom Payload)
* **LexEventDispatcher**: Utility class to define your intent handlers with enhanced processing hooks
* **IntentHandler**: Base class providing basic intent functionality with pre/post processing hooks
* **BaseIntentHandler**: Simplified base class for common intent patterns

### Enhanced Features (New in v2.0)
* **Rich Message Support**: SSML, Image Response Cards, Custom Payload messages
* **Runtime Hints**: Improve speech recognition accuracy with slot and phrase hints
* **Multi-valued Slots**: Handle List-shaped slots with multiple values
* **Sentiment Analysis**: Access user sentiment data from interpretations
* **Enhanced Slot Validation**: Built-in validators for common patterns (email, phone, etc.)
* **Context Management**: Utilities for managing active contexts and conversation state
* **Message Builders**: Fluent interfaces for creating complex messages
* **Slot Handlers**: Advanced slot manipulation and validation utilities

## Quick Install
```python
pip install amazon-lex-helper
```

## Basic Example
The *LexEventDispatcher* class provides an [observer](https://refactoring.guru/design-patterns/observer/python/example#:~:text=Observer%20is%20a%20behavioral%20design,that%20implements%20a%20subscriber%20interface.) approach to register intent handlers:

```python
from amazon_lex_helper import LexEventDispatcher
from BookHotelIntent import BookHotelIntent

def lambda_handler(event, context):
    lexEventDispatcher = LexEventDispatcher()
    lexEventDispatcher.subscribe(
        BookHotelIntent("BookHotel")
    )
    return lexEventDispatcher.dispatch(event)
```

## Enhanced Examples

### Rich Messages with SSML and Cards
```python
from amazon_lex_helper import BaseIntentHandler, MessageBuilder, SSMLBuilder
from amazon_lex_helper import LexResponse

class BookHotelIntent(BaseIntentHandler):
    def __init__(self):
        super().__init__("BookHotel", required_slots=["Location", "CheckInDate"])
    
    def fulfill_intent(self, lex):
        location = lex.get_slot_interpreted_value("Location")
        
        # Create rich response with SSML and card
        messages = MessageBuilder() \
            .add_plain_text(f"Great! I found hotels in {location}.") \
            .add_ssml(
                SSMLBuilder()
                .speak("Perfect choice!")
                .pause("500ms")
                .emphasis(location, level="strong")
                .speak("has amazing hotels.")
                .build()
            ) \
            .add_image_response_card(
                title=f"Hotels in {location}",
                subtitle="Best available rates",
                image_url="https://example.com/hotel.jpg",
                buttons=[
                    {"text": "Book Now", "value": "book"},
                    {"text": "See More", "value": "more"}
                ]
            ).build()
        
        return LexResponse.close_with_rich_messages(
            lex.get_session_attrs(),
            lex.get_intent(),
            {},
            messages
        )
```

### Runtime Hints for Better Speech Recognition
```python
from amazon_lex_helper import RuntimeHintsBuilder, LexResponse

class FlightBookingIntent(BaseIntentHandler):
    def process_request(self, lex):
        if lex.is_requesting_slot("Origin"):
            # Add runtime hints for airport codes
            hints = RuntimeHintsBuilder() \
                .add_slot_hint("Origin", ["JFK", "LAX", "ORD", "SFO"]) \
                .add_phrase_hints(["John F Kennedy", "Los Angeles International"]) \
                .build()
            
            messages = [{"contentType": "PlainText", "content": "Which airport?"}]
            
            return LexResponse.elicit_slot_with_rich_messages(
                lex, "Origin", messages, runtime_hints=hints
            )
        
        return super().process_request(lex)
```

### Multi-valued Slots
```python
from amazon_lex_helper import SlotHandler, SlotValidator

class OrderPizzaIntent(BaseIntentHandler):
    def validate_slots(self, lex):
        errors = {}
        
        # Handle multi-valued toppings slot
        if lex.is_multi_valued_slot("Toppings"):
            toppings = lex.get_slot_values_list("Toppings")
            
            # Validate count
            if len(toppings) > 5:
                errors["Toppings"] = "Please choose no more than 5 toppings."
            
            # Validate each topping
            valid_toppings = ["pepperoni", "mushrooms", "cheese"]
            for topping in toppings:
                if not SlotValidator.validate_choice(topping, valid_toppings):
                    errors["Toppings"] = f"{topping} is not available."
                    break
        
        return errors
```

### Sentiment Analysis and Enhanced Validation
```python
class CustomerServiceIntent(BaseIntentHandler):
    def handle_negative_sentiment(self, lex):
        """Handle frustrated customers with empathy."""
        messages = MessageBuilder() \
            .add_plain_text("I understand your frustration. Let me help you right away.") \
            .add_ssml(
                SSMLBuilder()
                .prosody("I'm here to help", rate="slow", volume="soft")
                .build()
            ).build()
        
        return LexResponse.elicit_intent_with_rich_messages(
            lex, "CustomerService", "InProgress", messages
        )
    
    def validate_slots(self, lex):
        errors = {}
        
        # Validate email with built-in validator
        email = lex.get_slot_interpreted_value("Email")
        if email and not SlotValidator.validate_email(email):
            errors["Email"] = "Please provide a valid email address."
        
        return errors
```

### Context Management
```python
from amazon_lex_helper import ContextManager, ConversationState

class ShoppingIntent(BaseIntentHandler):
    def __init__(self):
        super().__init__("Shopping")
        self.conversation_state = ConversationState("shoppingCart")
    
    def process_request(self, lex):
        contexts = lex.get_active_contexts()
        
        # Get current cart items
        cart_items = self.conversation_state.get_state_value(contexts, "items")
        
        if cart_items:
            message = f"You have {cart_items} in your cart. What else would you like?"
        else:
            message = "Welcome! What would you like to shop for today?"
        
        # Update conversation state
        contexts = self.conversation_state.update_state(contexts, {
            "last_interaction": "shopping_start"
        })
        
        response = LexResponse.elicit_intent_with_rich_messages(
            lex, "Shopping", "InProgress", [{"contentType": "PlainText", "content": message}]
        )
        response['sessionState']['activeContexts'] = contexts
        
        return response
```

## Available Utilities

### Message Builders
- `MessageBuilder`: Fluent interface for creating multiple message types
- `SSMLBuilder`: Create SSML content with prosody, emphasis, pauses
- `CardBuilder`: Create image response cards with buttons
- `RuntimeHintsBuilder`: Create runtime hints for speech recognition

### Slot Utilities
- `SlotHandler`: Create and manipulate scalar and list slots
- `SlotValidator`: Built-in validators (email, phone, date, numeric range, etc.)

### Context Utilities
- `ContextManager`: Manage active contexts and attributes
- `ConversationState`: Maintain conversation state across turns

### Quick Functions
```python
from amazon_lex_helper import quick_text_message, quick_ssml_message, quick_card_message

# Quick message creation
text_msg = quick_text_message("Hello!")
ssml_msg = quick_ssml_message("<speak>Hello <emphasis>world</emphasis>!</speak>")
card_msg = quick_card_message("Title", "Subtitle", buttons=[{"text": "OK", "value": "ok"}])
```

## Migration from v1.x

The library is backward compatible. Existing code will continue to work, but you can enhance it with new features:

1. **Replace `IntentHandler`** with `BaseIntentHandler` for simpler intent handling
2. **Add rich messages** using `MessageBuilder` and enhanced response functions
3. **Use runtime hints** for better speech recognition
4. **Access sentiment analysis** with `lex.get_sentiment_analysis()`
5. **Handle multi-valued slots** with `lex.get_slot_values_list()`

## AWS Lambda usage

You can clone this repo and execute ./create_layer.sh script, which will create a .zip file inside /layer folder.  
That zip can be then used to create a layer for your [AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/adding-layers.html).

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

