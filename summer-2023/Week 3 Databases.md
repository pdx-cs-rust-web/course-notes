- Homework Solutions
- Start talking about projects

## Finishing CRUD for questions

- Add Routes for question CRUD
```rust
pub fn get_router() -> Router {
    let db = AppDatabase::default();
    Router::new()
        .route("/questions", get(handlers::get_questions))

        // Question CRUD - Create/Read/Update/Delete
        .route("/question", post(handlers::create_question))
        .route("/question/:question_id", get(handlers::get_question_by_id))
        .route("/question", put(handlers::update_question))
        .route("/question", delete(handlers::delete_question))

        .route("/*_", get(handle_404))
        .with_state(db)
}
```

```rust handlers.rs
use axum::extract::{Path, Query, State};
use axum::Json;
use tracing::info;
use crate::db::AppDatabase;
use crate::error::{AppError, QuestionError};
use crate::question::{CreateQuestion, GetQuestionById, Question, QuestionId, UpdateQuestion};

pub async fn root() -> String {
    "Hello world!".to_string()
}

pub async fn get_questions(
    State(am_database): State<AppDatabase>
) -> Result<Json<Vec<Question>>, AppError> {
    let mut questions = am_database.questions.lock().unwrap();

    let db_count = questions.len() as usize;
    let question = Question::new(
        QuestionId(db_count),
        "Default question".to_string(),
        "Default Content".to_string(),
        Some(vec!("Default tag".to_string())),
    );

    (*questions).push(question.clone());

    let all_questions: Vec<Question> = (*questions).clone();

    Ok(Json(all_questions))
}

pub async fn get_question_by_id(
    State(am_database): State<AppDatabase>,
    Path(query): Path<u32>,
) -> Result<Json<Question>, AppError> {
    let questions = am_database.questions.lock().expect("Poisoned mutex");
    info!("Path is: {}", &query);

    let result_question = (*questions).iter().find(|&q| q.id.0 == query as usize);

    if let Some(question) = result_question {
        Ok(Json(question.clone()))
    } else {
        Err(AppError::Question(QuestionError::InvalidId))
    }
}

pub async fn create_question(
    State(am_database): State<AppDatabase>,
    Json(question): Json<CreateQuestion>,
) -> Result<Json<Question>, AppError> {
    let mut questions = am_database.questions.lock().expect("Poisoned mutex");
    let db_count = questions.len() as usize;
    let question_with_id = Question::new(
        QuestionId(db_count),
        question.content,
        question.title,
        question.tags,
    );
    // Must clone question to use it later
    (*questions).push(question_with_id.clone());

    // Send back the new question
    Ok(Json(question_with_id))
}

pub async fn update_question(
    State(am_database): State<AppDatabase>,
    Json(question): Json<UpdateQuestion>,
) -> Result<(), AppError> {
    let mut questions = am_database.questions.lock().expect("Poisoned mutex");

    let mut update = (*questions).get_mut(question.id.0).ok_or(AppError::Question(QuestionError::InvalidId))?;
    update.title = question.title;
    update.content = question.content;
    update.tags = question.tags;
    Ok(())
}

pub async fn delete_question(
    State(am_database): State<AppDatabase>,
    Query(query): Query<GetQuestionById>,
) -> Result<(), AppError> {
    let mut questions = am_database.questions.lock().expect("Poisoned mutex");

    questions.retain(|q| q.id.0 != query.question_id);

    Ok(())
}


```

## DATA STORES:

- We have a working backend now, but we have a big problem remaining.  Our "Database" code is mixed right into the middle of our route handlers.  What we'd like instead is for our handlers to be database agnostic -- if we want to swap our `Vec<Question>` for a "real" database, we should be able to do that without touching our handlers!
- For this, we use a modular services concept of a data Store.  We'll put all of our direct db-handling code into one of those, then our handlers will merely call the proper Store method!

-  We'll rename AppDatabase to Store to match the book
-  Add a new() fn
-  Add a fn for adding a new question
-  Add an init fn that adds some new questions


``` rust
use std::sync::{Arc, Mutex};
use anyhow::bail;
use crate::error::AppError;
use crate::question::{Question, QuestionId};

#[derive(Clone, Default)]
pub struct Store {
    pub questions: Arc<Mutex<Vec<Question>>>,
}

impl Store {
    pub fn new() -> Self {
        Store::default()
    }

    pub fn init(&mut self) -> Result<(), AppError> {
        let question = Question::new(
            QuestionId(0),
            "How?".to_string(),
            "Please help!".to_string(),
            Some(vec!["general".to_string()])
        );
        self.add_question(question)?;
        Ok(())
    }

    // Can we NOT clone here?  What would the implications be?
    // Arc<Vec<Question>>?  Arc<Mutex<Vec<Question>>>?
    pub fn get_all_questions(&self) -> Vec<Question> {
        self.questions.lock().unwrap().clone()
    }

    pub fn add_question(&mut self, title: String, content: String, tags: Option<Vec<String>>) -> Result<Question, AppError> {
        let mut questions = self.questions.lock().expect("Poisoned mutex");
        let len = questions.len();

        let new_question = Question {
            id: QuestionId(len as u32),
            title,
            content,
            tags,
        };
        questions.push(new_question.clone());
        Ok(new_question)
    }
}
```

## Handler updates

We can now update our get_questions handler to make use of our new Store!
``` rust
pub async fn get_questions(
    State(am_database): State<Store>
) -> Result<Json<Vec<Question>>, AppError> {
    let all_questions = am_database.get_all_questions();

    Ok(Json(all_questions))
}
```

# GetQuestionById
- We'll again want to move our code out of handler into our Store!
```rust
// Store
  pub fn get_question_by_id(&self, id: QuestionId) -> Result<Question, AppError> {
        let mut questions = self.questions.lock().expect("Poisoned mutex");
        let question = questions.iter().find(|q| q.id == id).cloned();

        question.ok_or(AppError::Question(QuestionError::InvalidId))
    }
// Handler
pub async fn get_question_by_id(
    State(am_database): State<Store>,
    Path(query): Path<u32>,
) -> Result<Json<Question>, AppError> {
   let question = am_database.get_question_by_id(query.into())?;

    Ok(Json(question))
}
```
- Now we're movin!  Notice that once we have everything organized into the right places, how incredibly easy it is to write code!

## CreateQuestion

- CreateQuestion we already have a Store fn for, since we used it for initialization.  We just need to fix our handler now

```rust
pub async fn create_question(
    State(mut am_database): State<Store>,
    Json(question): Json<CreateQuestion>,
) -> Result<Json<Question>, AppError> {
    let new_question = am_database.add_question(question.title, question.content, question.tags)?;

    // Send back the new question
    Ok(Json(new_question))
}
```

## UpdateQuestion

```rust
//Store
 pub fn update_question(&mut self, new_question: Question) -> Result<Question, AppError> {
        let mut questions = self.questions.lock().expect("Poisoned mutex");


        // Assuming the `id` of a `Question` is its index in the `Vec`.
        let index = new_question.id.0;

        // Check if the index is valid.
        if index < 0 || index as usize >= questions.len() {
            return Err(AppError::Question(QuestionError::InvalidId));
        }

        // Update the question at the given index.
        questions[index as usize] = new_question.clone();

        Ok(new_question)
    }
// Handler
pub async fn update_question(
    State(mut am_database): State<Store>,
    Json(question): Json<Question>,
) -> Result<Json<Question>, AppError> {
   let updated_question = am_database.update_question(question)?;
    Ok(Json(updated_question))
}
```

## DeleteQuestion

```rust
// Store
pub fn delete_question(&mut self, question_id: QuestionId) -> Result<(), AppError> {
        let mut questions = self.questions.lock().expect("Poisoned mutex");
        questions.retain(|q| q.id != question_id);

        Ok(())
    }
// Handler
pub async fn delete_question(
    State(mut am_database): State<Store>,
    Query(query): Query<GetQuestionById>,
) -> Result<(), AppError> {
    am_database.delete_question(query.question_id.into())?;

    Ok(())
}

```

- This is now a complete REST API for our Question entities!  Questions are a bit lame without Answers, though.  We could add entire CRUD routes and handlers and Data Store and such for these Answers, but shortly we'll be transitioning away from our Vector mock storage and using a real database, so for now, lets simply set up our Answer struct and the barest necessities.

- Create answer.rs

  ```rust
  use serde::{Deserialize, Serialize};
  use crate::question::QuestionId;
  
  #[derive(Serialize, Deserialize, Debug, Clone)]
  pub struct Answer {
      pub id: AnswerId,
      pub content: String,
      pub question_id: QuestionId,
  }
  #[derive(Deserialize, Serialize, Debug, Clone, PartialEq, Eq, Hash)]
  pub struct AnswerId(pub i32);
  
  impl From<i32> for AnswerId {
      fn from(value: i32) -> Self {
          Self(value)
      }
  }
  
  ```

- Add answers to our Store

  ```rust
  #[derive(Clone, Default)]
  pub struct Store {
      pub questions: Arc<Mutex<Vec<Question>>>,
      // Mutex is faster than RwLock (less overhead) but cannot allow multiple readers at once
      pub answers: Arc<RwLock<Vec<Answer>>>
  }
  //... impl Store
   pub fn add_answer(&mut self, content: String, question_id: QuestionId) -> Result<Answer, AppError> {
          let mut answers = self.answers.write().unwrap();
          let len = answers.len() as i32;
          let new_answer = Answer{
              id: len.into(),
              content,
              question_id,
          };
          answers.push(new_answer.clone());
          Ok(new_answer)
  
      }
  
  ```

- Add Route

  ```rust
     .route("/answer", post(handlers::create_answer))
  ```

- Add Handler

  ```rust
  pub async fn create_answer(
      State(mut am_database): State<Store>,
      Json(query): Json<CreateAnswer>
  ) -> Result<Json<Answer>, AppError> {
      let new_answer = am_database.add_answer(query.content, query.question_id)?;
      Ok(Json(new_answer))
  }
  
  ```

- Add CreateAnswer query struct

  ```rust
  #[derive(Debug, Serialize, Deserialize)]
  pub struct CreateAnswer {
      pub content: String,
      pub question_id: QuestionId
  }
  
  ```

- All done!  What are some issues we have now?

  - Deleting an item causes duplicate IDs because we're basing them only on vec size
  - If we supply a QuestionId for a new Answer that doesn't match any questions....it lets us!
  - We avoid these issues by using a real database!  

  ## Databases

- Book uses Postgres which is extra unfriendly to set up.  
- We'll not be writing any Docker ourselves in this course, but we'll use it for easy Postgres (Database) setup!

Linux (Debian) - Download a .deb package and install it: https://docs.docker.com/desktop/install/ubuntu/#install-docker-desktop

Windows - Install this .exe https://docs.docker.com/desktop/install/windows-install/

OSX - Install this .dmg https://docs.docker.com/desktop/install/mac-install/

- Again, this is ALL of the setup you'll need to do for this course!   We're going to use a pre-generated config file that will give us a happy Postgres database without any work


```yaml docker-compose.yaml
version: '3'

services:
  postgres:
    container_name: postgres
    image: postgres:15-alpine
    restart: always
    ports:
      - "5432:5432"
    volumes:
      - db:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=rustweb
      - POSTGRES_PASSWORD=rustweb
      - POSTGRES_DB=postgres

volumes:
  minio_data:
    driver: local
  db:
    driver: local
```

- Once you've added this `docker-compose.yaml` file to the root of your repository, ALL you need to do is execute `docker compose up` in that directory, then wait.  In a few minutes, your new Postgres is setup, running, and ready to connect to on port 5432

### Connecting To Your New DB

- We now have a database running!  We do not, however, yet know how to interact with it.  For *all* databases, whether Postgres or SQLite or Mongo, you will require a client.
https://wiki.postgresql.org/wiki/PostgreSQL_Clients

- You'll notice a big JETBRAINS IDEs on the list - We already have the finest database client ever made packed into this baby
- Beekeeper Studio looks very much like VS Code and works intuitively
- ANY of them is fine!  We'll not use it for very much other than checking the state of our database as we work in a quick way, so pick any that strikes your fancy.

## SQL

- We use one of the world's oldest still-in-daily-use languages, SQL (right behind COBOL/FORTRAN/Lisp!)

```sql
CREATE TABLE IF NOT EXISTS questions
(
    id         serial PRIMARY KEY,
    title      VARCHAR(255) NOT NULL,
    content    TEXT         NOT NULL,
    tags       TEXT[],
    created_on TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS answers
(
    id                     serial PRIMARY KEY,
    content                TEXT      NOT NULL,
    created_on             TIMESTAMP NOT NULL DEFAULT NOW(),
    corresponding_question integer REFERENCES questions
);

DROP TABLE answers;
DROP TABLE questions;

INSERT INTO questions(title, content, tags) VALUES ('Question Title', 'Question Content', ARRAY['tag1', 'tag2']);
INSERT INTO questions(title, content, tags) VALUES ('Another Title', 'Another Question Content', ARRAY['tag1', 'tag2']);

INSERT INTO answers(content, corresponding_question) VALUES ('Some answer content', 1);
INSERT INTO answers(content, corresponding_question) VALUES ('Some answer content', 17);

SELECT * from questions;

SELECT title, content from questions;

SELECT * from questions where id = 1;

SELECT * from answers;

SELECT * from questions, answers WHERE questions.id = answers.corresponding_question;

SELECT questions.id, title, content, tags from questions, answers WHERE questions.id = answers.corresponding_question;

SELECT questions.id, title, questions.content, answers.content, tags from questions, answers WHERE questions.id = answers.corresponding_question;

BEGIN;
INSERT INTO questions(title, content, tags) VALUES ('Another Title', 'Another Question Content', ARRAY['tag1', 'tag2']);
INSERT INTO answers(content, corresponding_question) VALUES ('Some answer content', (SELECT currval('questions_id_seq')));
COMMIT;

SELECT * from questions LEFT JOIN answers ON questions.id = answers.corresponding_question;
SELECT * from questions RIGHT JOIN answers ON questions.id = answers.corresponding_question;
SELECT * from questions INNER JOIN answers ON questions.id = answers.corresponding_question;
SELECT * from questions FULL OUTER JOIN answers ON questions.id = answers.corresponding_question;

```

