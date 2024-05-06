```
let dir_service = ServeDir::new(path)
    .append_index_html_on_directories(true)
    .precompressed_gzip()
    .precompressed_br()
    .precompressed_deflate()
    .precompressed_zstd()
    .fallback(ListDirService {
        state: Arc::new(state.clone()),
    });

#[derive(Debug, Clone)]
struct ListDirService {
    state: Arc<HttpServeState>,
}

impl<B> Service<axum::http::Request<B>> for ListDirService
where
    B: Send + 'static,
{
    type Response = Response;
    type Error = Infallible;
    type Future = futures::future::BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<std::result::Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: axum::http::Request<B>) -> Self::Future {
        let state = self.state.clone();
        let path = req.uri().path()[1..].to_string();

        Box::pin(async move {
            match list_dir(State(state), &path).await {
                Ok(response) => Ok(response),
                Err(status_code) => {
                    let response = Response::builder()
                        .status(status_code)
                        .body("".into())
                        .unwrap();
                    Ok(response)
                }
            }
        })
    }
}

```
