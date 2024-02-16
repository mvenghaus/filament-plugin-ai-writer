# Filament Plugin "AI Writer"

Unchain the power of AI in your Filament Panel! With this plugin you can easily let AI write all your content.

## Requirements

You need Filament v3, a valid license and an OpenAI API Token.

## Installation

To install the package via composer you need to add the private repository to your composer.json.

```json
"repositories": [
  {
    "type": "composer",
    "url": "https://filament-plugin-ai-writer.composer.sh"
  }
]

```

Now you are ready to install the package:

```bash
composer require mvenghaus/filament-plugin-ai-writer:"^3.0"
```

You will be asked for a username and password. Use your order email address as username and the license key you received as password.


## Configuration

Register the plugin in panel provider:

```php

use Mvenghaus\FilamentPluginAIWriter\FilamentPlugin;
use Mvenghaus\FilamentPluginAiWriter\Integration\OpenAI;
use Mvenghaus\FilamentPluginAiWriter\Integration\OpenAI\Models\Gpt4;

...

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ...
            ->plugin(
                FilamentPlugin::make(
                    // create the Open AI Integration
                    OpenAI::make(
                        'YOUR_TOKEN',
                        // choose the model you want to use
                        Gpt4::make(
                            maxTokens: 100 // optional                		
                        )
                    )
                )
            )
        }
    }

...

```

### Options

#### maxTokens
To limit your costs you can use the maxTokens option.

### OpenAI - Supported Models

- Mvenghaus\FilamentPluginAiWriter\Integration\OpenAI\Models\Gpt35Turbo
- Mvenghaus\FilamentPluginAiWriter\Integration\OpenAI\Models\Gpt4

## Usage

Basically the integration is "just" a button, but a very powerful one.

The principle is always structured as follows:

```
************       ************       ************
*  SOURCE  *  -->  *  MODIFY  *  -->  *  TARGET  *
************       ************       ************
```

### Basis Structure
```php
<?php

use Mvenghaus\FilamentPluginAiWriter\Forms\Components\AIWriterButton;

...

    public static function form(Form $form): Form
    {
        return $form
            ->schema([
                AIWriterButton::make('name')
                    ->label(...) // optional
                    ->source(...)
                    ->sourceModal(...) // optional
                    ->modify(...) // optional
                    ->target(...)
...

```

### Source

The source accpets a Closure and defines the request text that is send to the AI integration.
The Closure is evaluated like everything in Filament so you can use "Get", "Set", etc..

#### Predefined text

```php
->source(fn() => 'Tell me a joke about PHP')
```

#### Based on another field

```php
->source(fn(Get $get) => $get('source_field'))
```

#### Modal

Maybe you just want to generate a text on the fly without having any source text or field. There comes the "sourceModal" method in handy. 

```php
->sourceModal()
```

With this you get a modal with a textarea where you can enter your request text.

You can also combine it with the "source" method to get a prefilled modal form.

```php
->source(fn() => 'Prefilled Text')
->sourceModal()
```

### Modify

Modify also accepts a Closure as parameter. Here you have the possibility to manipulate the request text before it is send to the integration.
You get ```string $sourceText``` as param in your Closure.

This is for example useful when you want to work with placeholders in your request text.

Let's assume you have a form with a title and want to write a text with a predefined specification:

```php
->source(fn() => 'Write a text about "#title#" in 10 words')
->sourceModal()
->modify(
    fn(Get $get, string $sourceText) => str_replace('#title#', $get('title'), $sourceText)
)
```

### Target

At last there is the target. Here you decide what you want to do with the generated text. Here you also work with a Closure.
As Closure parameter you get a response data object.

```php
/** @var \Mvenghaus\FilamentPluginAIWriter\Data\Response $reponse */
$response->text // string - the generated text
$response->information // array - additional information from the integration response 
```

Writing the text to a field would look like this:

```php
->source(fn(Get $get) => $get('source_field'))
->target(fn(Set $set, Response $response) => $set('target_field', $response->text))
```

## Examples

With this flexible structure you can do a lot of awesome things. Let's dive into some examples.

### Source Field -> Target Field

**Code**

```php
AIWriterButton::make('generate')
    ->source(fn(Get $get) => $get('source_field'))
    ->target(fn(Set $set, Response $response) => $set('target_field', $response->text))
```

**Demo**

![Demo](https://raw.githubusercontent.com/mvenghaus/filament-plugin-ai-writer/main/images/source_field_target_field.gif)


### Appending text using modal

**Code**

```php
AIWriterButton::make('generate')
    ->label('Add Content')
    ->sourceModal()
    ->target(function(Get $get, Set $set, Response $response) {
        $set(
            'target_field',
            trim(sprintf("%s\n\n%s", $get('target_field'), $response->text))
        );
    }),
```

**Demo**

![Demo](https://raw.githubusercontent.com/mvenghaus/filament-plugin-ai-writer/main/images/append_text_using_modal.gif)

### Translate field

**Code**

```php
AIWriterButton::make('translate')
    ->source(fn(Get $get) => 'translate in german: '. $get('description')))
    ->target(fn(Set $set, Response $response) => $set('description', $response->text))
```

**Demo**
![Demo](https://raw.githubusercontent.com/mvenghaus/filament-plugin-ai-writer/main/images/translate_field.gif)

### Product Description

**Code**

```php
AIWriterButton::make('generate')
    ->source(function(Get $get) {
        return implode("\n", [
            'write a product description:',
            'Name: ' . $get('name'),
            'Type: ' . $get('type'),
            'Color: ' . $get('color'),
            'Sizes: ' . $get('sizes'),
        ]);
    })
    ->target(fn(Set $set, Response $response) => $set('description', $response->text)),
```

**Demo**
![Demo](https://raw.githubusercontent.com/mvenghaus/filament-plugin-ai-writer/main/images/product_description.gif)

## Writing your own AI Integration

Currently only Open AI is supported. But you can easily write your own integration.
To achive this you simply have to implement the "Integration" contract.

```php
<?php

use Mvenghaus\FilamentPluginAIWriter\Contracts\Integration;
use Mvenghaus\FilamentPluginAIWriter\Data\Response;

class MyIntegration implements Integration
{
    public function request(string $text): Response
    {
        // call your ai provider
        $result = (...);
        
        return new Response(
            content: $result
        );
    }
}
```

And then pass your Integration in the admin panel provider:

```php

use Mvenghaus\FilamentPluginAIWriter\FilamentPlugin;

...

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ...
            ->plugin(
                FilamentPlugin::make(new MyIntegration())
            )
        }
    }

...

```

## Contact
If you any questions or you find a bug, please let me now at [Github](https://github.com/mvenghaus/filament-plugin-ai-writer/issues) or reach out on Discord.
