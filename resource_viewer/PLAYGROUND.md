# Playground — All Supported Markdown Features

This document demonstrates every rendering feature supported by the Resource Viewer. Each feature shows the raw markdown syntax followed by the rendered result.

---

## Headings

```
# Heading 1
## Heading 2
### Heading 3
#### Heading 4
##### Heading 5
###### Heading 6
```

# Heading 1
## Heading 2
### Heading 3
#### Heading 4
##### Heading 5
###### Heading 6

---

## Text Formatting

```
**bold text**, *italic text*, ***bold and italic***
~~strikethrough~~ and `inline code`
```

**bold text**, *italic text*, ***bold and italic***

~~strikethrough~~ and `inline code`

---

## Links

### Standard Markdown Links

```
[PromptFerret](https://promptferret.github.io)
[Back to Welcome](WELCOME.md)
[Jump to Tables section](#tables)
```

[PromptFerret](https://promptferret.github.io)

[Back to Welcome](WELCOME.md)

[Jump to Tables section](#tables)

### Obsidian Wikilinks

```
[[WELCOME]]
[[WELCOME|Back to the welcome page]]
```

[[WELCOME]]

[[WELCOME|Back to the welcome page]]

---

## Images

### External Image

```
![Placeholder](https://promptferret.github.io/assets/promptferret_small.jpg)
```

![Placeholder](https://promptferret.github.io/assets/promptferret_small.jpg)

### Relative Image

```
![My Image](images/promptferret.jpeg)
```

![My Image](images/promptferret.jpeg)

Relative images resolve from the document's folder. The above would load `images/promptferret.jpeg` relative to this document.

---

## Code

### Inline Code

```
Use `console.log()` to print output.
```

Use `console.log()` to print output.

### Fenced Code Blocks

````
```javascript
function greet(name) {
  const message = `Hello, ${name}!`;
  console.log(message);
  return message;
}
```
````

```javascript
function greet(name) {
  const message = `Hello, ${name}!`;
  console.log(message);
  return message;
}
```

````
```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b
```
````

```python
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b
```

````
```css
.document {
  max-width: 800px;
  margin: 0 auto;
  color: var(--text);
}
```
````

```css
.document {
  max-width: 800px;
  margin: 0 auto;
  color: var(--text);
}
```

````
```
Plain code block — no language specified.
  Indentation is preserved.
```
````

```
Plain code block — no language specified.
  Indentation is preserved.
```

---

## Lists

### Unordered Lists

```
- First item
- Second item with **bold** and `code`
- Third item
```

- First item
- Second item with **bold** and `code`
- Third item

```
* Alpha
* Beta
* Gamma
```

* Alpha
* Beta
* Gamma

### Ordered Lists

```
1. First step
2. Second step with *emphasis*
3. Third step
```

1. First step
2. Second step with *emphasis*
3. Third step

---

## Blockquotes

```
> This is a simple blockquote.
```

> This is a simple blockquote.

```
> This blockquote has **bold text**, *italics*, and `inline code` inside it.
```

> This blockquote has **bold text**, *italics*, and `inline code` inside it.

```
> Multiple lines in a blockquote
> continue on the next line
> as long as they start with >
```

> Multiple lines in a blockquote
> continue on the next line
> as long as they start with >

---

## Callouts

### Note

```
> [!note] A Note
> This is a note callout. Use it for general information.
```

> [!note] A Note
> This is a note callout. Use it for general information.

### Abstract / TLDR

```
> [!abstract] Abstract
> A brief summary of the content that follows.
```

> [!abstract] Abstract
> A brief summary of the content that follows.

```
> [!tldr] Too Long; Didn't Read
> The tldr alias maps to the abstract style.
```

> [!tldr] Too Long; Didn't Read
> The tldr alias maps to the abstract style.

### Info / Todo

```
> [!info] Information
> Useful background context or reference material.
```

> [!info] Information
> Useful background context or reference material.

```
> [!todo] Todo Item
> The todo alias maps to the info style.
```

> [!todo] Todo Item
> The todo alias maps to the info style.

### Tip / Hint / Important

```
> [!tip] Pro Tip
> A helpful suggestion to improve your workflow.
```

> [!tip] Pro Tip
> A helpful suggestion to improve your workflow.

```
> [!hint] Hint
> The hint alias maps to the tip style.
```

> [!hint] Hint
> The hint alias maps to the tip style.

```
> [!important] Important
> The important alias also maps to the tip style.
```

> [!important] Important
> The important alias also maps to the tip style.

### Success / Check / Done

```
> [!success] Success
> The operation completed successfully.
```

> [!success] Success
> The operation completed successfully.

```
> [!done] Done
> The done alias maps to the success style.
```

> [!done] Done
> The done alias maps to the success style.

### Question / Help / FAQ

```
> [!question] Frequently Asked
> How do callouts work? Use the `> [!type] Title` syntax.
```

> [!question] Frequently Asked
> How do callouts work? Use the `> [!type] Title` syntax.

```
> [!faq] FAQ
> The faq alias maps to the question style.
```

> [!faq] FAQ
> The faq alias maps to the question style.

### Warning / Caution / Attention

```
> [!warning] Warning
> Be careful — this operation may have side effects.
```

> [!warning] Warning
> Be careful — this operation may have side effects.

```
> [!caution] Caution
> The caution alias maps to the warning style.
```

> [!caution] Caution
> The caution alias maps to the warning style.

### Failure / Fail / Missing

```
> [!failure] Failure
> The operation did not complete as expected.
```

> [!failure] Failure
> The operation did not complete as expected.

```
> [!missing] Missing
> The missing alias maps to the failure style.
```

> [!missing] Missing
> The missing alias maps to the failure style.

### Danger / Error

```
> [!danger] Danger
> This action is irreversible and may cause data loss.
```

> [!danger] Danger
> This action is irreversible and may cause data loss.

```
> [!error] Error
> The error alias maps to the danger style.
```

> [!error] Error
> The error alias maps to the danger style.

### Bug

```
> [!bug] Known Bug
> There is a known issue with this feature.
```

> [!bug] Known Bug
> There is a known issue with this feature.

### Example

```
> [!example] Example Usage
> Here is how you would use this feature in practice.
```

> [!example] Example Usage
> Here is how you would use this feature in practice.

### Quote / Cite

```
> [!quote] Quote
> "The best way to predict the future is to invent it." — Alan Kay
```

> [!quote] Quote
> "The best way to predict the future is to invent it." — Alan Kay

### Summary

```
> [!summary] In Summary
> This covers all 13 callout types and their aliases.
```

> [!summary] In Summary
> This covers all 13 callout types and their aliases.

### Collapsible Callouts

```
> [!tip]- Click to Expand
> This callout starts collapsed. Click the header to toggle.
> It supports **formatted text** and `code` inside.
```

> [!tip]- Click to Expand
> This callout starts collapsed. Click the header to toggle.
> It supports **formatted text** and `code` inside.

```
> [!warning]- Another Collapsed Callout
> Use the `-` modifier after the type: `[!type]-`
```

> [!warning]- Another Collapsed Callout
> Use the `-` modifier after the type: `[!type]-`

### Callout Without Title

```
> [!note]
> If no title is provided, only the icon is shown in the header.
```

> [!note]
> If no title is provided, only the icon is shown in the header.

### Statblock (D&D Parchment)

````
> [!statblock] Shadow Mastiff
> *Medium monstrosity, Typically Neutral Evil*
>
> **Armor Class** 12
> **Hit Points** 33 (6d8 + 6)
> **Speed** 40 ft.
>
> **STR** 16 (+3) **DEX** 14 (+2) **CON** 13 (+1) **INT** 5 (−3) **WIS** 12 (+1) **CHA** 5 (−3)
>
> **Skills** Perception +5, Stealth +6
> **Damage Resistances** bludgeoning, piercing, slashing from nonmagical attacks while in dim light or darkness
> **Senses** darkvision 60 ft., passive Perception 15
> **Challenge** 2 (450 XP)
>
> ---
>
> ### Traits
>
> **Ethereal Awareness.** The shadow mastiff can see ethereal creatures and objects.
>
> **Sunlight Weakness.** While in bright light created by sunlight, the shadow mastiff has disadvantage on attack rolls, ability checks, and saving throws.
>
> ---
>
> ### Actions
>
> **Bite.** *Melee Weapon Attack:* +5 to hit, reach 5 ft., one target. *Hit:* 10 (2d6 + 3) piercing damage. If the target is a creature, it must succeed on a DC 13 Strength saving throw or be knocked prone.
>
> ---
>
> ### Bonus Actions
>
> **Shadow Blend.** While in dim light or darkness, the shadow mastiff becomes invisible, along with anything it is wearing or carrying. The invisibility lasts until the shadow mastiff uses a bonus action to end it or until the shadow mastiff attacks, is in bright light, or is incapacitated.
````

> [!statblock] Shadow Mastiff
> *Medium monstrosity, Typically Neutral Evil*
>
> **Armor Class** 12
> **Hit Points** 33 (6d8 + 6)
> **Speed** 40 ft.
>
> **STR** 16 (+3) **DEX** 14 (+2) **CON** 13 (+1) **INT** 5 (−3) **WIS** 12 (+1) **CHA** 5 (−3)
>
> **Skills** Perception +5, Stealth +6
> **Damage Resistances** bludgeoning, piercing, slashing from nonmagical attacks while in dim light or darkness
> **Senses** darkvision 60 ft., passive Perception 15
> **Challenge** 2 (450 XP)
>
> ---
>
> ### Traits
>
> **Ethereal Awareness.** The shadow mastiff can see ethereal creatures and objects.
>
> **Sunlight Weakness.** While in bright light created by sunlight, the shadow mastiff has disadvantage on attack rolls, ability checks, and saving throws.
>
> ---
>
> ### Actions
>
> **Bite.** *Melee Weapon Attack:* +5 to hit, reach 5 ft., one target. *Hit:* 10 (2d6 + 3) piercing damage. If the target is a creature, it must succeed on a DC 13 Strength saving throw or be knocked prone.
>
> ---
>
> ### Bonus Actions
>
> **Shadow Blend.** While in dim light or darkness, the shadow mastiff becomes invisible, along with anything it is wearing or carrying. The invisibility lasts until the shadow mastiff uses a bonus action to end it or until the shadow mastiff attacks, is in bright light, or is incapacitated.

---

## Tables

### Simple Table

```
| Name | Role | Level |
|---|---|---|
| Thorn | Fighter | 5 |
| Elara | Wizard | 5 |
| Pip | Rogue | 4 |
```

| Name | Role | Level |
|---|---|---|
| Thorn | Fighter | 5 |
| Elara | Wizard | 5 |
| Pip | Rogue | 4 |

### Table with Inline Formatting

```
| Feature | Syntax |
|---|---|
| Bold | **text** |
| Italic | *text* |
| Code | `code` |
| Link | [text](url) |
| Strikethrough | ~~text~~ |
```

| Feature | Syntax |
|---|---|
| Bold | **text** |
| Italic | *text* |
| Code | `code` |
| Link | [text](url) |
| Strikethrough | ~~text~~ |

---

## Horizontal Rules

```
---
```

---

```
***
```

***

```
___
```

___

---

## Mixed Content

Callouts support full block-level content — lists, tables, and code blocks all work inside them.

````
> [!example]- Mixed Content Inside a Callout
> Callouts can contain **bold**, *italic*, `code`, ~~strikethrough~~, and [links](https://example.com).
>
> - Item one
> - Item two
> - Item three
>
> | Column A | Column B |
> |---|---|
> | Value 1 | Value 2 |
>
> ```
> console.log('Code blocks work inside callouts too!');
> ```
````

> [!example]- Mixed Content Inside a Callout
> Callouts can contain **bold**, *italic*, `code`, ~~strikethrough~~, and [links](https://example.com).
>
> - Item one
> - Item two
> - Item three
>
> | Column A | Column B |
> |---|---|
> | Value 1 | Value 2 |
>
> ```
> console.log('Code blocks work inside callouts too!');
> ```

---

*End of playground — all supported features demonstrated above.*
