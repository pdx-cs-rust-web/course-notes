JWTs

We need 3 new features now 

1. Ability to create a new user with a secure password
2. Ability to login as a user
3. Ability to authenticate already-logged-in users



## Tokens

https://jwt.io/introduction

1. **What is JWT?**

   JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with the HMAC algorithm) or a public/private key pair using RSA or ECDSA.

2. **Structure of JWT**

   A JWT typically looks like `xxxxx.yyyyy.zzzzz` and is composed of three parts separated by dots (`.`):

   - **Header**: The header typically consists of two parts: the type of the token, which is JWT, and the signing algorithm being used, such as HMAC SHA256 or RSA.
   - **Payload**: The second part of the token is the payload, which contains the claims. Claims are statements about an entity (typically, the user) and additional data. There are three types of claims: registered, public, and private claims.
   - **Signature**: To create the signature part, you have to take the encoded header, the encoded payload, a secret, the algorithm specified in the header, and sign that.

   Hence, a JWT typically looks like this: `base64UrlEncode(header) + '.' + base64UrlEncode(payload) + '.' + signature`.

3. **How JWTs Work?**

   In authentication, when the user successfully logs in using their credentials, a JSON Web Token will be returned. Since tokens are credentials, great care must be taken to prevent security issues. In general, you should not keep tokens longer than required.

   Whenever the user wants to access a protected route or resource, the user agent should send the JWT, typically in the `Authorization` header using the `Bearer` schema. The content of the header might look like the following: `Authorization: Bearer <token>`.

   This is a stateless authentication mechanism as the user state is never saved in server memory. The server's protected routes will check for a valid JWT in the `Authorization` header, and if it's present, the user will be allowed to access protected resources.

   If the token is sent in the `Authorization` header, Cross-Origin Resource Sharing (CORS) won't be an issue as it doesn't use cookies.

4. **Why use JWTs?**

   JSON Web Tokens offer a compact, URL-safe means of representing claims to be transferred between two parties. They are used for:

   - **Authorization**: Once the user is logged in, each subsequent request will include the JWT, allowing the user to access routes, services, and resources that are permitted with that token.
   - **Information Exchange**: JSON Web Tokens are a good way of securely transmitting information between parties. Because JWTs can be signed—for example, using public/private key pairs—you can be sure the senders are who they say they are.

In conclusion, JWTs provide a means of maintaining session state on the client, where the token itself is inherently stateless. 

## JWTs in Rust

https://docs.rs/jsonwebtoken/latest/jsonwebtoken/

https://github.com/Keats/jsonwebtoken



## Our goal flow

### Registration

1) User attempts to register by sending email/password/confirm_password
2) We check to see if the passwords match
3) We check to see if there's already a user with that email address existing
4) Create the account by adding new row to User table

### Login

1. User sends email/password
2. We check to ensure a user with that email exists
3. We check to ensure their password matches
4. We create a JWT with their information
5. We send the JWT back to the client



### Authentication

1. We look for JWT claims attached to the request
2. We decrypt and validate the JWT claims
3. We send the user their results if they were properly tokenized

## Refactor

Lets move our models to a /models subdir and our handlers to /handlers and routes to /routes.  Soon we'll break these all apart, so this will make it easier later



## Users

In order to impl any auth at all, we need a users table - sqlx migrate add -r add_users_table

```
CREATE TABLE IF NOT EXISTS users
(
    id         serial PRIMARY KEY,
    email      VARCHAR(255) NOT NULL,
    password   VARCHAR(255)         NOT NULL,
    created_on TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

......

-- Add down migration script here
DROP TABLE users;

```

- [ ] Run migrations



## SETUP

We'll need a few package changes and a new .env entry

```
// cargo.toml
once_cell = "1.18"
axum = { version = "0.6.2", features = ["headers", "multipart"] }
```

- [ ] .env addition secret

  ```
  JWT_SECRET=veryverysecretpasswordnolooking
  ```

  

- [ ] New error type additions:

  ```
  #[derive(Debug)]
  pub enum AppError {
      Question(QuestionError),
      Database(sqlx::Error),
      MissingCredentials,
      UserDoesNotExist,
      UserAlreadyExists,
      InvalidToken,
      InternalServerError,
      #[allow(dead_code)]
      Any(anyhow::Error),
  }
  
  
  
  impl IntoResponse for AppError {
      fn into_response(self) -> Response {
          let (status, error_message) = match self {
              AppError::Question(err) => match err {
                  QuestionError::InvalidId => (StatusCode::NOT_FOUND, err.to_string()),
              },
              AppError::Database(err) => (StatusCode::SERVICE_UNAVAILABLE, err.to_string()),
              AppError::Any(err) => {
                  let message = format!("Internal server error! {}", err);
                  (StatusCode::INTERNAL_SERVER_ERROR, message)
              }
              AppError::MissingCredentials => {
                  (StatusCode::UNAUTHORIZED, "Your creds were missing or incorrect".to_string())
              }
              AppError::UserDoesNotExist => {
                  (StatusCode::UNAUTHORIZED, "User does not exist".to_string())
              }
              AppError::UserAlreadyExists => { (StatusCode::UNAUTHORIZED, "User already exists".to_string())}
              AppError::InternalServerError => {
                  let message = format!("Internal server error!");
                  (StatusCode::INTERNAL_SERVER_ERROR, message)
              }
              AppError::InvalidToken => (StatusCode::UNAUTHORIZED, "Invalid Token".to_string())
          };
  
          let body = Json(json!({"error": error_message}));
          (status, body).into_response()
      }
  }
  ```

  

## Registration

Lets add a new POST route to /users for registering

```
  .route("/users", post(handlers::register))
```

And we'll need some structs to represent the different User states

```
// user.rs
use axum::extract::{FromRequest, FromRequestParts};
use axum::headers::Authorization;
use axum::headers::authorization::Bearer;
use jsonwebtoken::{decode, DecodingKey, EncodingKey, Validation};
use once_cell::sync::Lazy;
use serde::{Deserialize, Serialize};
use crate::error::AppError;
use axum::{async_trait, RequestPartsExt, TypedHeader};
use derive_more::Display;
use http::request::Parts;


#[derive(Serialize, Deserialize, sqlx::FromRow)]
pub struct User {
    pub email: String,
    pub password: String,
}

#[derive(Serialize, Deserialize, sqlx::FromRow)]
pub struct UserSignup {
    pub email: String,
    pub password: String,
    pub confirm_password: String
}

#[derive(Serialize, Deserialize)]
pub struct LoggedInUser {
    pub token: Claims,
}

#[derive(Deserialize, Serialize, Display)]
#[display(
fmt = "email: {}, exp: {}",
email,
exp

)]
pub struct Claims {
    pub email: String,
    pub exp: u64,
}

pub struct Keys {
    pub encoding: EncodingKey,
    pub decoding: DecodingKey,
}

impl Keys {
    pub fn new(secret: &[u8]) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
        }
    }
}

pub static KEYS: Lazy<Keys> = Lazy::new(|| {
    let secret = std::env::var("JWT_SECRET").unwrap_or_else(|_| "Your secret here".to_owned());
    Keys::new(secret.as_bytes())
});





```



And a handler for it:

```rust
pub async fn register(
    State(mut am_database): State<Store>,
    Json(credentials): Json<UserSignup>,
) -> Result<Json<Value>, AppError> {
    // check if email or password is a blank string
    if credentials.email.is_empty() || credentials.password.is_empty() {
        return Err(AppError::MissingCredentials);
    }

    if credentials.password != credentials.confirm_password {
        return Err(AppError::MissingCredentials);
    }

    let user = am_database.get_user(&credentials.email).await;

    if let Ok(_) = user {
        //if a user with email already exits send error
        return Err(AppError::UserAlreadyExists);
    }

    let new_user = am_database.create_user(credentials).await?;
    Ok(new_user)
}
```

And we'll need a database portion too:

```
   pub async fn get_user(&self, email: &str) -> Result<User, AppError>{
        let user = sqlx::query_as::<_, User>(
            r#"
    SELECT email, password FROM users WHERE email = $1
    "#,
        )
            .bind(email)
            .fetch_one(&self.conn_pool)
            .await?;

        Ok(user)
    }
 
 pub async fn create_user(&self, user: UserSignup) -> Result<Json<Value>, AppError> {
        let result = sqlx::query("INSERT INTO users (email, password) values ($1, $2)")
            .bind(&user.email)
            .bind(user.password)
            .execute(&self.conn_pool)
            .await
            .map_err(|_| AppError::InternalServerError)?;
        if result.rows_affected() < 1 {
            Err(AppError::InternalServerError)
        } else {
            Ok(Json(serde_json::json!({ "msg": "registered successfully" })))
        }
    }
```

- Now we can test it with an http request

  ```
  ###
  POST http://localhost:3000/users
  Content-Type: application/json
  
  {
  "email": "email332@email.com",
  "password": "password",
  "confirm_password": "password"
  }
  ```

  

## Logging in

- We're going to need almost hte same setup as registration, starting with a new route

  ```
   .route("/login", post(handlers::login))
  ```

- And we'll need a handler

  ```
  pub async fn login(
      State(mut am_database): State<Store>,
      Json(credentials): Json<User>,
  ) -> Result<Json<Value>, AppError> {
      // check if email or password is a blank string
      if credentials.email.is_empty() || credentials.password.is_empty() {
          return Err(AppError::MissingCredentials);
      }
  
      // get the user for the email from database
      let user = am_database.get_user(&credentials.email).await?;
      
      // if password is encrypted than decode it first before comparing
      if user.password != credentials.password {
          // password is incorrect
          Err(AppError::MissingCredentials)
      } else {
          let claims = Claims {
              email: credentials.email.to_owned(),
              exp: get_timestamp_8_hours_from_now(),
          };
          let token = encode(&Header::default(), &claims, &KEYS.encoding)
              .map_err(|_| AppError::MissingCredentials)?;
          // return bearer token
          Ok(Json(json!({ "access_token": token, "type": "Bearer" })))
      }
  }
  
  .....
  // lib.rs
  pub fn get_timestamp_8_hours_from_now() -> u64 {
      let now = SystemTime::now();
      let since_the_epoch = now.duration_since(UNIX_EPOCH).expect("Time went backwards");
      let eighthoursfromnow = since_the_epoch + Duration::from_secs(28800);
      eighthoursfromnow.as_secs()
  }
  ```

- We can now also test this one, with another http request

  ```
  ###
  POST http://localhost:3000/login
  Content-Type: application/json
  
  {
    "email": "email332@email.com",
    "password": "password"
  }
  ```

  

## Protecting Routes

- Now we'll need a way to protect routes from entry for people with invalid tokens.  We can do this by building our very own Axum extractor, which will fish teh JWT out of a request and provide it on-demand to a handler

- Lets add a new route to protect, such that only people with a JWT token that validates can come to it:

  ```
          .route("/protected", get(handlers::protected))
          
          ...
          
          pub async fn protected(claims: Claims) -> Result<String, AppError> {
      // Send the protected data to the user
      Ok(format!(
          "Welcome to the protected area :)\nYour data:\n{}",
          claims
      ))
  }
  
  ```

  

  ```
  #[async_trait]
  impl<S> FromRequestParts<S> for Claims
      where
          S: Send + Sync,
  {
      type Rejection = AppError;
  
      async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
          // Extract the token from the authorization header
          let TypedHeader(Authorization(bearer)) = parts
              .extract::<TypedHeader<Authorization<Bearer>>>()
              .await
              .map_err(|_| AppError::InvalidToken)?;
          // Decode the user data
          let token_data = decode::<Claims>(bearer.token(), &KEYS.decoding, &Validation::default())
              .map_err(|_| AppError::InvalidToken)?;
  
          Ok(token_data.claims)
      }
  }
  ```

  

- And once again, we'll test these with a request:

  ```
  ###
  GET http://localhost:3000/protected
  Content-Type: application/json
  Accept: application/json
  Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImVtYWlsMzMyQGVtYWlsLmNvbSIsImV4cCI6MTY5MDg1OTg1Nn0.9P4OVgaWN1rV2uw-ADNXxexwPgYk2Ec1-52dINIvh9U
  
  
  ```

  