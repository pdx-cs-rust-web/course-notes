# Client - HTML Templating

## Housekeeping

- Look through HW2/repo changes/additions on Github

- Talk about
- Older method that arguably should be used far more often today
- Modern frameworks are extreme overkill for simple sites
- All we really need is the ability to display some HTML with a tiny bit of dynamic data
- Jinja is the most well known of these -- https://documentation.bloomreach.com/engagement/docs/jinja-syntax
- Most Rust templating crates take from Jinja, so we will too!



## Creating our own templating engine

- We'll only need a single dependency, our old friend `regex`
- We need only a few features
  - 1) Print the value of variables `<body> Hi {{ name}} </body>`
    2) Loops `{% repeat 3 %}` ... `{% end %}`
    3) If statements `{% if a_bool %}`
    4) Comments `{# List of questions #}`

- We'll also need a `render()` function that does the converting.

  ```
  fn render<Stringly: AsRef<str>>(mut template: Stringly, mut data: HashMap<&str, Data>) -> String {
  
  ```

  - Here, template is our HTML itself, and `data` is a hashmap full of our various template substitutions

    - ```rs
      HashMap::from([
          ("name", Data::Text("bob".to_string())),
      ])
      ```

    - Data is an enum we'll create to wrap each Rust type that we want to be able to convert to HTML string text.

```rs
enum Data {
    Number(i32),
    Boolean(bool),
    Text(String)
}
impl fmt::Display for Data {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Self::Text(x) => write!(f, "{}", x),
            Self::Number(x) => write!(f, "{}", x),
            Self::Boolean(x) => write!(f, "{}", x)
        }
    }
}

```



### Feature 1 - print

- Printing variable values is the prime utility of a templating engine
- We'll first need to identify our {{ }} pattern.  For this, we'll use Regex!
- RegExr makes these wonderfully easy to test
- `let print_regex = Regex::new(r"\{\{(.*?)\}\}").unwrap();` (note the escaped \)
  - `{{` — Match opening double braces
  - `(.*?)` — Match any text [lazily](https://javascript.info/regexp-greedy-and-lazy#lazy-mode)
  - `{{` — Match closing double braces
- Now we want to iterate each of the matches and replace w/ value that matches our key

```rust
template = print_regex.replace_all(&template, |caps: &Captures| {
    ...
    // Find the corresponding key in the Data and return the value
    data[key].to_string()
}).to_string();
```

- These captures always start with the full string, so caps[0] = "{{ hello }}"

- Caps[1] will include the spaces, so we'll want to strip them!

- ```rs
  // Extract the text in between the {{ and the }} in the template
  let key = caps.get(1).unwrap().as_str().trim();
  ```

## Feature 2 - Loops

- We'll use a different regex for this
- `let repeat_regex = Regex::new(r"\{% repeat (\d*?) %\}((.|\n)*?)\{% end %\}").unwrap();`
  - `{% repeat ` — Match first part of opening tag
  - `(\d*?)` — Match digits(s) as capture group
  - ` %}` — Match end part of opening tag
  - `((.|\n)*?)` — Match multi-line string of text -- this is what we're repeating
  - `{% endrepeat %}` — Match closing tag

```rs
template = repeat_regex.replace_all(&template, |caps: &Captures| {
    // Extract the number of times to repeat the code
    let times = caps.get(1).unwrap().as_str().trim();

    // Parse the code block to be repeated
    let code = caps.get(2).unwrap().as_str().trim();

    // Repeat the code `times` number of times
    code.repeat(times.parse::<usize>().unwrap())
}).to_string();
```



## Feature 3 - If statements

- We're now looking to match `{% if BOOL %}...{% else %}...{% endif %}`

- `let if_else_regex = Regex::new(r"\{% if (.*?) %\}((.|\n)*?)(\{% else %\}((.|\n)*?)\{% endif %\}|\{% endif %\})").unwrap();`

  - `{% if ` — Match first part of opening tag
  - `(.*?)` — Match any text (the boolean to test)
  - ` %}` — Match end part of opening tag
  - `((.|\n)*?)` — Match multi-line string of text (If block)
  - `{% else %}` — Match else tag
  - `((.|\n)*?)` — Match multi-line string of text (Else block)
  - `{% endif %}` — Match closing tag

- simply use the `regex` crate's [`.replace_all()`](https://docs.rs/regex/latest/regex/struct.Regex.html#method.replace_all) method to swap the capture with either the parsed `if` code block or the parsed `else` code block, depending on the value of the boolean that matches `key`

- ```Rust
  template = if_else_regex.replace_all(&template, |caps: &Captures| {
      // Extract the name of the bool being tested
      let key = caps.get(1).unwrap().as_str().trim();
      // Parse the 'if' and (optional) 'else' code blocks
      let if_code = caps.get(2).unwrap().as_str().trim();
      let else_code = caps.get(5).map_or("", |m| m.as_str()).trim();
      // Find the corresponding key in the Data and return the value
      if let Data::Boolean(exp) = data[key] {
          if exp { if_code.to_string() }
          else { else_code.to_string() }
      } else {
          "ERROR PARSING KEY".to_string()
      }
  }).to_string();
  
  ```



## Feature 4 - Comments

- Finally we can skip the regex and directly replace these!
- template = template.replace("{#", "<!--").replace("#}", "-->");



## Finished Render fn

```rust
fn render(mut template: String, mut data: HashMap<&str, Data>) -> String {
    // Render variable printing
    let print_regex = Regex::new(r"\{\{(.*?)\}\}").unwrap();
    template = print_regex.replace_all(&template, |caps: &Captures| {
        // Extract the text in between the {{ and the }} in the template
        let key = caps.get(1).unwrap().as_str().trim();
        // Find the corresponding key in the Data and return the value
        data[key].to_string()
    }).to_string();

    // Render repeat statements
    let repeat_regex = Regex::new(r"\{% repeat (\d*?) %\}((.|\n)*?)\{% end %\}").unwrap();
    template = repeat_regex.replace_all(&template, |caps: &Captures| {
        // Extract the number of times to repeat the code
        let times = caps.get(1).unwrap().as_str().trim();
        // Parse the code block to be repeated
        let code = caps.get(2).unwrap().as_str().trim();
        // Repeat the code `times` number of times
        code.repeat(times.parse::<usize>().unwrap())
    }).to_string();

    // Render for statements
    let if_else_regex = Regex::new(r"\{% if (.*?) %\}((.|\n)*?)(\{% else %\}((.|\n)*?)\{% endif %\}|\{% endif %\})").unwrap();
    template = if_else_regex.replace_all(&template, |caps: &Captures| {
        // Extract the name of the bool being tested
        let key = caps.get(1).unwrap().as_str().trim();
        // Parse the 'if' and (optional) 'else' code blocks
        let if_code = caps.get(2).unwrap().as_str().trim();
        let else_code = caps.get(5).map_or("", |m| m.as_str()).trim();
        // Find the corresponding key in the Data and return the value
        if let Data::Boolean(exp) = data[key] {
            if exp { if_code.to_string() }
            else { else_code.to_string() }
        } else {
            "ERROR PARSING KEY".to_string()
        }
    }).to_string();

    // Process comments
    template = template.replace("{#", "<!--").replace("#}", "-->");

    // Return output
    template
}

```



## Writing a template of our own!

- Lets create a `templates/` subdir to hold our templates

- ```
  <body>
      {# This is a comment! #}
      <h1>{{ hello }}</h1>
  
      {% if allowed %}
          {% repeat 3 %}
              <p>Welcome to the {{ hello }}!</p>
          {% end %}
      {% else %}
          <p>No trespassing.</p>
      {% endif %}
  
      {#
      <p>Hidden from the {{ hello }}...</p>
      #}
  </body>
  
  ```



## Testing it

```#[cfg(test)]
mod tests {
    use crate::{render, Data};
    use std::collections::HashMap;

    #[test]
    fn basic_template() {
        let input = std::fs::read_to_string("templates/index.html").expect("Something went wrong reading the file");
		let data = HashMap::from([
    ("hello", Data::Text("IT WORKED".to_string())),
    ("allowed", Data::Boolean(true))
]);
println!("{}", render(input, data));

        assert!(result.find("WORKED").is_some())
    }
}
```



## Swapping to a "real" templating engine

- Our engine has a LOT of weaknesses and missing functionality, so now that we've explored how templating works, lets use a real one, Tera - https://github.com/Keats/tera
- We can find some fine axum-specific examples to build from! https://github.com/Altair-Bueno/axum-template
- <<Fill in notes after class with whatever we build>>