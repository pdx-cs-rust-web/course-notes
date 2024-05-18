## Client

1. Next we'll use Reqwest to make these requests to our backend.  We'll create a client project as well
`cargo new --bin client`

```toml
[dependencies]
anyhow = "1.0"
reqwest = "0.11"
serde = { version = "1.0", features = ["derive"] }
serde_derive = "1.0"
serde_json = "1.0.96"
tokio = { version = "1", features = ["full"] }
```

```rust
use reqwest::Client;

#[tokio::main]
async fn main() -> anyhow::Result<()>{
    // Create a reqwest client
    let client = Client::new();

    //Make a GET HTTP request to our backend's /example route
    let res = client.get("http://localhost:3000/questions").send().await?;

    //Get the response from backend's data
    let body = res.text().await?;
    //Print out that response
    println!("GET Response: {}", body);

    Ok(())
}

```

## AXUM
- We're cleaning house!  We'll get rid of EVERYTHING in backend and convert to axum
- New deps list
```toml
[dependencies]
anyhow = "1.0"
axum = "0.6.2"
axum-macros = "0.3.1"
axum-derive-error = "0.1.0"
backtrace = "0.3.67"
bcrypt = "0.14.0"
dotenvy = "0.15.6"
chrono = { version = "0.4.10", features = ["serde"] }
derive_more = "0.99.2"
futures = "0.3.1"
header = "0.1.1"
html-escape = "0.2.13"
http = "0.2.9"
http-serde = "1.1.2"
hyper = "0.14.26"
jsonwebtoken = "8.0.1"
mime = "0.3.17"
r2d2 = "0.8.8"
rand = "0.8.5"
reqwest = { version = "0.11.13", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_derive = "1.0"
serde_json = "1.0"
sqlx = { version = "0.6", features = ["runtime-tokio-rustls", "postgres"] }
termcolor = "1.2.0"
time = "0.3.20"
tokio = { version = "1.0", features = ["full"] }
thiserror = "1.0"
tower-http = { version = "0.4.0", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
uuid = { version = "1.3.0", features = ["serde", "v4"] }
```

- Setup tokio, dotenv, and logging.  We'll also add a run fn
```rust
use std::{
    net::{IpAddr, SocketAddr},
    str::FromStr,
};
use axum::Router;
use axum::routing::get;

use dotenvy::dotenv;
use tracing::{debug, info};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // Collect environment variables from a .env file
    dotenv().ok();
    // Initialize tokio's logging/tracing
    init_logging();
    debug!("Application initialized.");


    // This runs our listen server
    run().await;
}

async fn run() {
	// Get the host:port combo from our .env file
    let addr = get_host_from_env();

    info!("Listening on {}", addr);

	// Create our mapping between path/method and function
    let app = Router::new()
        .route("/questions", get(|| async { "Hello world!!!!"}));
    // Start the server!
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .expect("Rust server died, must be hardware issue");
}

fn init_logging() {
    // https://github.com/tokio-rs/axum/blob/main/examples/tracing-aka-logging
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                // axum logs rejections from built-in extractors with the `axum::rejection`
                // target, at `TRACE` level. `axum::rejection=trace` enables showing those events
                "backend=trace,tower_http=debug,axum::rejection=trace".into()
            }),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();
}

// Basic utility fn for creating a Rusty-address from .env vars
fn get_host_from_env() -> SocketAddr {
    let host = std::env::var("API_HOST").expect("Could not find API_HOST from environment");
    let api_host =
        IpAddr::from_str(&host).expect("Could not create viable IP Address from API_HOST");
    let api_port: u16 = std::env::var("API_PORT")
        .expect("Could not find API_PORT from environment")
        .parse()
        .expect("Could not create viable port from API_PORT environment variable.");

    SocketAddr::from((api_host, api_port))
}
```
- [ ] Create new files for organization `touch db.rs error.rs handlers.rs layers.rs question.rs routes.rs`
- [ ] Create .env file
```
API_HOST=127.0.0.1
API_PORT=3000
```
- Next we need ourselves a fancy fake database `db.rs`
```rust db.rs
use std::sync::{Arc, Mutex};
use crate::Question;

// Mock database only
#[derive(Clone, Default)]
pub struct AppDatabase {
    pub questions: Arc<Mutex<Vec<Question>>>,
}
```

- Next, lets clean up our code a bit.  We'll put question into its own file `question.rs`
```rust
use std::fmt;

use derive_more::Display;
use serde_derive::{Deserialize, Serialize};

// This uses the `derive_more` crate to reduce the Display boilerplate (see below)
#[derive(Clone, Debug, Display, Serialize, Deserialize)]
#[display(
fmt = "id: {}, title: {}, content: {}, tags: {:?}",
id,
title,
content,
tags
)]
pub struct Question {
    pub id: QuestionId,
    pub title: String,
    pub content: String,
    pub tags: Option<String>,
}

impl Question {
    pub fn new(id: QuestionId, title: String, content: String, tags: Option<String>) -> Self {
        Question {
            id,
            title,
            content,
            tags,
        }
    }
}

#[derive(Clone, Debug, Display, Serialize, Deserialize)]
pub struct QuestionId(pub usize);

// Clients use this to create new requests
#[derive(Debug, Serialize, Deserialize)]
pub struct CreateQuestion {
    pub title: String,
    pub content: String,
    pub tags: Option<String>,
}

#[derive(Deserialize)]
pub struct GetQuestionById {
    pub question_id: usize,
}

/* Manual impl (without derive_more)
impl std::fmt::Display for Question {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        write!(
            f,
            "{}, title: {}, content: {}, tags: {:?}",
            self.id, self.title, self.content, self.tags
        )
    }
}

impl std::fmt::Display for QuestionId {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        write!(f, "id: {}", self.0)
    }
}
*/


```
- That Router is going to get mighty big, mighty fast!
- [ ] Create routes.rs
```rust routes.rs
use axum::response::Response;
use axum::Router;
use axum::routing::{delete, get, post};
use http::StatusCode;
use hyper::Body;
use crate::db::AppDatabase;

pub fn get_router() -> Router {
    let db = AppDatabase::default();
    Router::new()
        .route("/", get(handlers::root))
        .route("/questions", get(handlers::get_questions))
        .route("/question", get(handlers::get_question_by_id))
        .route("/question", post(handlers::create_question))
        .route("/question", delete(handlers::delete_question))
        // This is our catchall, MUST be at the end!
        .route("/*_", get(handle_404))
        .with_state(db)
}

async fn handle_404() -> Response<Body> {
    Response::builder()
        .status(StatusCode::NOT_FOUND)
        .body(Body::from("The requested resource could not be found"))
        .unwrap()
}
```
- We'll next need to build an Error type for our server.  Axum will intercept these errors and return them as an HTTP Response for us!
```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;
use derive_more::Display;
use serde_json::json;
use thiserror::Error;

/// Our app's top level error type.
#[derive(Error, Debug, Display)]
pub enum AppError {
    // Specific Error types
    Question(QuestionError),
    // Catchall Error route for anyhow bails
    Any(anyhow::Error),
}

// Axum's main error handler apparatus - we simply turn them into Responses!
// Note here that we're seeing a "Successful Failure" here, which HTTP considers a success
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // Fish a Status Code and error message out of each error
        let (status, error_message) = match self {

            AppError::Question(err) => match err {
                QuestionError::InvalidId => (StatusCode::NOT_FOUND, err.to_string()),
            },
            // No touchy!  Handles *all* anyhow-originating errors
            AppError::Any(err) => {
                let message = format!("Internal Server Error: {}", err);
                (StatusCode::INTERNAL_SERVER_ERROR, message)
            }
        };

        let body = Json(json!({ "error": error_message }));
        (status, body).into_response()
    }
}

// Allows `?` to automatically convert all anyhow Errors into AppErrors
impl From<anyhow::Error> for AppError {
    fn from(inner: anyhow::Error) -> Self {
        Self::Any(inner)
    }
}

#[derive(Debug, Error)]
pub enum QuestionError {
    #[error("Question ID Not Found")]
    InvalidId,
}

impl From<QuestionError> for AppError {
    fn from(inner: QuestionError) -> Self {
        Self::Question(inner)
    }
}

```
- Now we'll need to add what are called `layers`.  They alter requests BEFORE landing in our handler and in other frameworks are normally called `middleware`
- We're going to use two, so we'll make a new `layers.rs` file 
- Our important one is CORS:
- When developing APIs that are available to the wider public, you need to think about
cross-origin resource sharing (CORS); see http://mng.bz/N5wD. Web browsers have a
security mechanism that doesn’t allow requests started from domain A against domain
B. For your development and production requirements, deploying a website to a
server, and trying to send requests from your browser locally or from a different
domain, will fail.

Here's an example of an attack that CORS can help prevent:

1. Assume that you're logged into your bank's website (bank.com) and you have a session cookie stored in your browser.
2. You then visit a malicious site (evil.com) that, unbeknownst to you, tries to use JavaScript to send a request to bank.com to transfer money to an attacker's account. Without any kind of same-origin policy in place, the browser would happily send along your session cookie, the bank's website would assume that the request was legitimate and coming from you, and the transfer would take place.
3. This is where CORS comes into play. By implementing CORS, the bank's server can set specific headers that allow only certain origins (like bank.com itself) to make requests. When the malicious site (evil.com) tries to make the request, the browser checks the server's CORS headers, sees that evil.com isn't a trusted origin, and blocks the request from going through.

For this scenario, CORS was invented. It should soften the stance on the same-origin
policy (http://mng.bz/DDwE) and allow browsers to make requests to other domains.
How do they do it? Instead of sending, for example, an HTTP PUT request directly,
they send a preflight request to the server, which is an HTTP OPTIONS request. Figure
3.5 shows what is involved in the preflight request.
This OPTIONS request asks the server if it’s OK to send the request, and the
server replies with the allowed methods in the header. The browser reads the allowed
methods, and if PUT is included, it does a second HTTP request with the actual data
in the body.
```rust layers.rs
use http::Method;
use tower_http::classify::{ServerErrorsAsFailures, SharedClassifier};
use tower_http::cors::{Any, CorsLayer};
use tower_http::trace::TraceLayer;

pub fn get_layers() -> (
    CorsLayer,
    // Don't let this scare you!  This is what makes our Trace understand that Axum errors are "Failures" and thus it will report them
    TraceLayer<SharedClassifier<ServerErrorsAsFailures>>,
) {
    let cors_layer = CorsLayer::new()
        //.allow_origin(addr.to_string().parse::<HeaderValue>().unwrap()) // Proper CORS handling
        .allow_origin(Any) // for convenience
        .allow_headers([http::header::CONTENT_TYPE])
        .allow_methods([
            Method::GET,
            Method::POST,
            Method::PUT,
            Method::DELETE,
            Method::OPTIONS,
        ]);

    let trace_layer = TraceLayer::new_for_http();
    (cors_layer, trace_layer)
}

```

```rust
// Setup cors and trace layer
async fn run() {
    let addr = get_host_from_env();

    info!("Listening on {}", addr);

    let (cors_layer, trace_layer) = layers::get_layers();

    let app = routes::get_router().layer(cors_layer).layer(trace_layer);
    // Start the server!
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .expect("Rust server died, must be hardware issue");
}
```

- Now we're only left with our handlers!
```rust handlers.rs
use axum::extract::{Query, State};
use axum::Json;

use shared::question::{CreateQuestion, GetQuestionById, Question, QuestionId};

use crate::db::AppDatabase;
use crate::http_error::{AppError, QuestionError};

// Basic hello-world route
pub async fn root() -> String {
    "Hello World!".into()
}

pub async fn get_questions(
    State(am_database): State<AppDatabase>,
) -> Result<Json<Vec<Question>>, AppError> {
    let mut questions = am_database.questions.lock().expect("Poisoned mutex");
    let db_count = questions.len() as usize;
    let question = Question::new(
        QuestionId(db_count),
        "Default Question".to_string(),
        "Default Content".to_string(),
        Some(vec!["default".to_string()]),
    );

    (*questions).push(question.clone());

    let all_questions: Vec<Question> = (*questions).clone();

    // This will currently always be true, of course
    #[allow(irrefutable_let_patterns)] //example code
    if let _ = question.id.0 as i32 {
        // If our question was found, we send it to the client
        Ok(Json(all_questions))
    } else {
        // Otherwise we send our error instead.
        // This will auto-populate the message for us
        Err(AppError::Question(QuestionError::InvalidId))
    }
}

pub async fn get_question_by_id(
    State(am_database): State<AppDatabase>,
    Query(query): Query<GetQuestionById>,
) -> Result<Json<Question>, AppError> {
    let questions = am_database.questions.lock().expect("Poisoned mutex");


    let result_question = (*questions).iter().find(|&q| q.id.0 == query.question_id);

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

pub async fn delete_question(
    State(am_database): State<AppDatabase>,
    Query(query): Query<GetQuestionById>,
) -> Result<(), AppError> {
    let mut questions = am_database.questions.lock().expect("Poisoned mutex");

    questions.retain(|q| q.id.0 == query.question_id);
    Ok(())
}

```