# lazers-hyper-client

A CouchDB client implemented using hyper.

This is currently a draft implementation that suffers from a few problems,
mainly that generating the errors hooking into lazers-traits a bit noisy.

This crate itself holds no logic outside of HTTP handling, the description
of
all workflows is in lazers-traits.


```rust
extern crate hyper;
extern crate url;
extern crate lazers_traits;
extern crate serde;
extern crate serde_json;
extern crate mime;
extern crate backtrace;

mod types;
use types::document_created::DocumentCreated;
use types::error;

use lazers_traits::prelude::*;

use serde_json::de::from_reader;
use serde_json::ser::to_string;

use hyper::header::ETag;
use hyper::header::ContentType;

use hyper::client::IntoUrl;

use hyper::status::StatusCode;
use std::sync::Arc;

use url::{Url, ParseError};

pub struct HyperClient {
    inner: hyper::client::Client,
    base_url: Url,
}

impl HyperClient {
    pub fn new<T: IntoUrl>(url: T) -> std::result::Result<HyperClient, ParseError> {
        Ok(HyperClient {
            inner: hyper::client::Client::new(),
            base_url: try!(url.into_url()),
        })
    }
}

impl Default for HyperClient {
    fn default() -> HyperClient {
        HyperClient {
            inner: hyper::client::Client::new(),
            base_url: Url::parse("http://localhost:5984").expect("this is a valid URL"),
        }
    }
}

pub struct RemoteDatabaseCreator {
    name: DatabaseName,
    base_url: Url,
}

pub struct RemoteDatabase {
    name: DatabaseName,
    base_url: Url,
}

impl DatabaseCreator for RemoteDatabaseCreator {
    type D = RemoteDatabase;

    fn create(self) -> Result<RemoteDatabase> {
        let mut url = self.base_url.clone();
        url.set_path(self.name.as_ref());
        let client = hyper::client::Client::new();
        let res = client.put(url)
            .send();
        try!(res.chain_err(|| self.name.clone()));

        Ok(RemoteDatabase {
            name: self.name,
            base_url: self.base_url,
        })
    }
}

impl Database for RemoteDatabase {
    type Creator = RemoteDatabaseCreator;

    fn destroy(self) -> Result<RemoteDatabaseCreator> {
        let mut url = self.base_url.clone();
        url.set_path(self.name.as_ref());
        let client = hyper::client::Client::new();
        let res = client.delete(url)
            .send();

        try!(res.chain_err(|| self.name.clone()));

        Ok(RemoteDatabaseCreator {
            name: self.name,
            base_url: self.base_url,
        })
    }

    fn doc<'a, K: Key, D: Document>(&'a self,
                                    key: K)
                                    -> Result<DatabaseEntry<'a, K, D, RemoteDatabase>> {
        let mut url = self.base_url.clone();
        url.set_path(format!("{}/{}", self.name, key.id()).as_ref());
        let client = hyper::client::Client::new();
        let res = client.get(url)
            .send();

        match res {
            Ok(r) => {
                match r.status {
                    StatusCode::Ok => {
                        let rev = r.headers.get::<ETag>().unwrap().clone();
                        let key_with_rev = <K as Key>::from_id_and_rev(key.id().to_owned(),
                                                                       Some(rev.tag().to_owned()));
                        let doc = from_reader(r).unwrap();
                        Ok(DatabaseEntry::present(key_with_rev, doc, self))
                    }
                    StatusCode::NotFound => Ok(DatabaseEntry::absent(key, self)),
                    _ => {
                        error(format!("Unexpected status: {}", r.status),
                              backtrace::Backtrace::new())
                    }
                }
            }
            Err(e) => {
                hyper_error(format!("Unexpected HTTP error"),
                            e,
                            backtrace::Backtrace::new())
            }
        }
    }

    // this should probably be &doc, as Doc won't be changed, but might
    // get a new key
    fn insert<K: Key, D: Document>(&self, key: K, doc: D) -> Result<(K, D)> {
        println!("{:?}", key);
        let mut url = self.base_url.clone();
        url.set_path(format!("{}/{}", self.name, key.id()).as_ref());

        if let Some(rev) = key.rev() {
            url.query_pairs_mut().append_pair("rev", rev);
        }

        let client = hyper::client::Client::new();
        let body = match to_string(&doc) {
            Ok(s) => s,
            Err(e) => {
                return hyper_error(format!("Unexpected HTTP error"),
                                   e,
                                   backtrace::Backtrace::new())
            }
        };

        let mime: mime::Mime = "application/json".parse().unwrap();
        let res = client.put(url)
            .header(ContentType(mime))
            .body(&body)
            .send();
        match res {
            Ok(r) => {
                match r.status {
                    StatusCode::Created => {
                        let response_data: DocumentCreated = from_reader(r).unwrap();

                        let k = K::from_id_and_rev(response_data.id, Some(response_data.rev));

                        Ok((k, doc))
                    }
                    StatusCode::Conflict => {
                        let response_data: error::Error = from_reader(r).unwrap();
                        match response_data {
                            error::Error::Conflict(reason) => {
                                conflict(format!("Document update conflict: {}", reason),
                                         backtrace::Backtrace::new())
                            }
                            error::Error::BadRequest(reason) => {
                                error(format!("Bad request: {}", reason),
                                      backtrace::Backtrace::new())
                            }
                        }
                    }
                    _ => {
                        error(format!("Unexpected status: {}", r.status),
                              backtrace::Backtrace::new())
                    }
                }
            }
            Err(e) => {
                hyper_error(format!("Unexpected HTTP error"),
                            e,
                            backtrace::Backtrace::new())
            }
        }
    }

    fn delete<K: Key>(&self, key: K) -> Result<()> {
        let mut url = self.base_url.clone();
        url.set_path(format!("{}/{}", self.name, key.id()).as_ref());
        url.query_pairs_mut().append_pair("rev", key.rev().unwrap());
        let client = hyper::client::Client::new();
        let res = client.delete(url)
            .send();

        match res {
            Ok(r) => {
                match r.status {
                    StatusCode::Ok => Ok(()),
                    _ => {
                        error(format!("Unexpected status: {}", r.status),
                              backtrace::Backtrace::new())
                    }
                }
            }
            Err(e) => {
                hyper_error(format!("Unexpected HTTP error"),
                            e,
                            backtrace::Backtrace::new())
            }
        }
    }
}

impl Client for HyperClient {
    type Database = RemoteDatabase;

    fn find_database(&self,
                     name: DatabaseName)
                     -> Result<DatabaseState<RemoteDatabase, RemoteDatabaseCreator>> {
        let mut url = self.base_url.clone();
        url.set_path(name.as_ref());
        let res = self.inner
            .head(url)
            .send();

        match res {
            Ok(r) => {
                match r.status {
                    StatusCode::Ok => {
                        Ok(DatabaseState::Existing(RemoteDatabase {
                            name: name,
                            base_url: self.base_url.clone(),
                        }))
                    }
                    StatusCode::NotFound => {
                        Ok(DatabaseState::Absent(RemoteDatabaseCreator {
                            name: name,
                            base_url: self.base_url.clone(),
                        }))
                    }
                    _ => {
                        error(format!("Unexpected status: {}", r.status),
                              backtrace::Backtrace::new())
                    }
                }
            }
            Err(e) => {
                hyper_error(format!("Unexpected HTTP error"),
                            e,
                            backtrace::Backtrace::new())
            }
        }
    }
}

fn hyper_error<T, E: std::error::Error + Send + 'static>(message: String,
                                                         error: E,
                                                         backtrace: backtrace::Backtrace)
                                                         -> Result<T> {
    Err(Error(ErrorKind::ClientError(message),
              (Some(Box::new(error)), Arc::new(backtrace))))
}

fn error<T>(message: String, backtrace: backtrace::Backtrace) -> Result<T> {
    Err(Error(ErrorKind::ClientError(message), (None, Arc::new(backtrace))))
}

fn conflict<T>(message: String, backtrace: backtrace::Backtrace) -> Result<T> {
    Err(Error(ErrorKind::ClientError(message), (None, Arc::new(backtrace))))
}
```
