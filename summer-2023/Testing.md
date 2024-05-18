# Testing



- The most important and awful topic of all!  We put it off for a while, but we should've done this from the start.

- Rust has *the* nicest testing!  If you're feeling a terrible sense of dread, don't!  Prepare to be amazed at the power of Rust.

### Unit Tests

Unit tests are small tests which verify the functionality of a specific section of code, typically at the level of a single function or module.

In Rust, unit tests are usually placed inside the module they're testing and are marked with the `#[test]` attribute. The `assert!` macro is often used to ensure that certain conditions are met.

Here's an example:

```rust
fn add_two(a: i32) -> i32 {
    a + 2
}

#[test]
fn test_add_two() {
    assert_eq!(4, add_two(2));
}
```

In this example, `add_two` is a simple function that we're testing with `test_add_two`. `assert_eq!` checks that the output of `add_two(2)` is `4`. If it's not, the test will fail.

### Running Tests

Tests can be run using the `cargo test` command in your terminal. Cargo compiles and runs all the tests in your project.

### Test Organization

The convention is to keep unit tests in the same files as the code they're testing, using a `tests` module marked with `#[cfg(test)]` attribute.

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_two() {
        assert_eq!(4, add_two(2));
    }
}
```

The `#[cfg(test)]` attribute tells Rust to only compile and run the test code if we're currently trying to run tests, not in the final compiled output.

### Integration Tests

For more complex testing scenarios, such as when you want to test how multiple parts of your code interact, you can create integration tests. These are typically placed in a `tests` directory at the top level of your project. Each file in the `tests` directory is compiled as a separate crate.

For example, if you have a library crate named `backend`, as we do, you could have a `tests` folder with a file named `integration_tests.rs` with content like this:

```rust
// in backend db.rs
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// integration_tests.rs
#[test]
fn test_greet() {
    let result = greeter::greet("Alice");
    assert_eq!(result, "Hello, Alice!");
}

```

- Lets try this by adding an integration test for `add_question` next.

## Web specific test concerns	

- We have some problems though!  Tests are built into Rust.  What is *NOT* built into Rust? 
- Tokio Test!  This attr gives us async capability in our tests.  It is quite hidden and I only found it by specifically looking for it....in Axum  https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs



- Lets add an integration test for our Hello World route, which we'll need to add to routes.rs

  ```rust
  #[tokio::test]
  async fn test_add_question() {
      let mut app = get_router().await;
  
      let question = CreateQuestion {
          title: "New Title".into(),
          content: "Test content2".into(),
          tags: None
      };
  
      let response = app
          .oneshot(
              Request::builder()
                  .method(http::Method::POST)
                  .uri("/question")
                  .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                  .body(Body::from(
                      serde_json::to_string(&question).unwrap()
                  ))
                  .unwrap(),
          )
          .await
          .unwrap();
  
      dbg!("{}", &response);
      assert_eq!(response.status(), StatusCode::OK);
  }
  ```

  

- REFACTOR TIME! Commit first!  We can't access anything from this test!  Lib vs main ruh roh.
- Hooray, it works!  However, we still have an extreme issue -- these tests are affecting our dev database.  Directly. Even when they fail.  This is A) Dangerous, and B) really annoying.  The only way to avoid the danger is to painstakingly delete everything currently IN the database, perform our migrations, perform our tests against it, then repeat, over and over.  Otherwise we run the chance of continually allowing tiny bugs in because we're writing code against NOT the database we think we are.  It's not overwhelmingly dangerous to the codebase itself, but it will sap your developer time.
- FS people will be shuddering in horror right about now, remembering the crazy lengths and terrible libraries we had to dig around in to get around this problem in JS.  Fear not, for SQLx arrives to save us!

https://docs.rs/sqlx/latest/sqlx/attr.test.html

### Dependency Injection

We now need to stop creating our own DB connection inside of Store, and instead give Store the DB connection we want it to use.  This is because SQLx will provide our "Test" database connection here during tests rather than using our development connection.  

- It creates an entire test database that exactly matches our schema...by using those handy-dandy migrations we built *automatically*  
- It does this for *each* sqlx::test independently, so our tests don't interfere with one another in addition to not interfering with our dev database



We'll need to refactor a bit more to make this one possible, but not too much! Commit again reminder.

- And now we can finally write a fully async, auto-migrated integration test

  ```
  #[sqlx::test]
  async fn test_add_question(db_pool: PgPool) {
  
      let mut app = app(db_pool).await;
  
      let question = CreateQuestion {
          title: "New Title".into(),
          content: "Test content2".into(),
          tags: None
      };
  
      let response = app
          .oneshot(
              Request::builder()
                  .method(http::Method::POST)
                  .uri("/question")
                  .header(http::header::CONTENT_TYPE, mime::APPLICATION_JSON.as_ref())
                  .body(Body::from(
                      serde_json::to_string(&question).unwrap()
                  ))
                  .unwrap(),
          )
          .await
          .unwrap();
  
      dbg!("{}", &response);
      assert_eq!(response.status(), StatusCode::OK);
  }
  
  ```

- This has even more awesome properties than on the surface.  If a test FAILS, SQLx will *leave* that test database temporarily, so you can poke through it to debug!  As soon as that test passes, the database will be cleaned up.

- But wait, we're missing one more thing!  Our test database has all of the structures/schemas, but no data.  We could copy our dev database, but that one's probably changing pretty often as we work.  What would be nicer would be to start from a guaranteed known position

## Fixtures

- For this, sqlx provides us yet another amazing setup.  We call these prior-construction facilities "Fixtures" and sqlx will load them for us by default.  

- Inside of `/tests` create another subdir `fixtures`, into which we'll put `questions.sql`

- We'll grab the INSERT queries from the SQL notes and put them here, then add a few more carefully constructed for testing

  ```sql
  INSERT INTO questions(title, content, tags) VALUES ('TestTitle1', 'Question Content', ARRAY['tag1', 'tag2']);
  INSERT INTO questions(title, content, tags) VALUES ('TestTitle2', 'Another Question Content', ARRAY['tag1', 'tag2']);
  INSERT INTO questions(title, content, tags) VALUES ('TestTitle3', 'Question Content', ARRAY['tag1', 'tag2']);
  INSERT INTO questions(title, content) VALUES ('TestTitle4', 'Another Question Content');
  	
  ```

  

- Now we'll expose that to each test we want it in individually `#[sqlx::test(fixtures("questions"))]`

- That is everything we'll need all in one tidy package!

## Users

- Lets reuse this entire flow to add a Users table in prep for all the fun authentication to come soon

```
CREATE TABLE IF NOT EXISTS users
(
    id                     serial PRIMARY KEY,
    email                VARCHAR(255)      NOT NULL, -- 255 is SMTP max length
    password         VARCHAR(255) NOT NULL,
    created_on             TIMESTAMP NOT NULL DEFAULT NOW()
);

```

- We can also add a create_user and login_user endpoint WITH TESTS to practice!