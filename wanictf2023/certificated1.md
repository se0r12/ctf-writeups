## certificated1

Rustはなにも読めませんが、感覚で読みます。

Router的なのは多分ここ。

*main.rs*
```rust
    let app = Router::new()
        .route("/", get(handler::index::handle_index))
        .route("/create", post(handler::create::handle_create))
        .route("/view/:id", get(handler::view::handle_view))
        .layer(TraceLayer::new_for_http());
```

ファイルをアップロードすると、`/create`へのリクエストが飛ぶので`handle_create`を見てみる。

*create.rs*
```rust
#[debug_handler]
pub async fn handle_create(mut multipart: extract::Multipart) -> HandlerResult {
    let id = Uuid::new_v4();

    let current_dir = PathBuf::from(format!("./data/{id}"));
    fs::create_dir(&current_dir)
        .await
        .context("Failed to create working directory")?;

    let (file_name, file_data) = match extract_file(&mut multipart).await {
        Some(file) => file,
        None => return Ok((StatusCode::BAD_REQUEST, "Invalid multipart data").into_response()),
    };
    fs::write(
        current_dir.join(file_name.file_name().unwrap_or("".as_ref())),
        file_data,
    )
    .await
    .context("Failed to save uploaded file")?;

    process_image(&current_dir, &file_name)
        .await
        .context("Failed to process image")?;

    Ok((StatusCode::SEE_OTHER, [("location", format!("/view/{id}"))]).into_response())
}
```

また、アップロードしたファイルには`承認　ワニ博士`というスタンプが押されるので、これがどこで行われるのかを見ると、`process_image`っぽい。

*process_image.rs*
```rust
pub async fn process_image(working_directory: &Path, input_filename: &Path) -> Result<()> {
    fs::copy(
        working_directory.join(input_filename),
        working_directory.join("input"),
    )
    .await
    .context("Failed to prepare input")?;

    fs::write(
        working_directory.join("overlay.png"),
        include_bytes!("../assets/hanko.png"),
    )
    .await
    .context("Failed to prepare overlay")?;

    let child = Command::new("sh")
        .args([
            "-c",
            "timeout --signal=KILL 5s magick ./input -resize 640x480 -compose over -gravity southeast ./overlay.png -composite ./output.png",
        ])
        .current_dir(working_directory)
        .stdin(Stdio::null())
        .stdout(Stdio::null())
        .stderr(Stdio::piped())
        .spawn()
        .context("Failed to spawn")?;

    let out = child
        .wait_with_output()
        .await
        .context("Failed to read output")?;

    if !out.status.success() {
        bail!(
            "image processing failed on {}:\n{}",
            working_directory.display(),
            std::str::from_utf8(&out.stderr)?
        );
    }

    Ok(())
}
```

はんこをつけているのは以下か？

*process_image.rs*
```rust
    let child = Command::new("sh")
        .args([
            "-c",
            "timeout --signal=KILL 5s magick ./input -resize 640x480 -compose over -gravity southeast ./overlay.png -composite ./output.png",
        ])
        .current_dir(working_directory)
        .stdin(Stdio::null())
        .stdout(Stdio::null())
        .stderr(Stdio::piped())
        .spawn()
        .context("Failed to spawn")?;
```

ImageMagickを使っていそう。

Dockerfileからversionは7.1.0-51を使っていそうなことがわかる。

*Dockerfile*
```text
ARG MAGICK_URL="https://github.com/ImageMagick/ImageMagick/releases/download/7.1.0-51/ImageMagick--gcc-x86_64.AppImage"
```

[CVE-2022-44268](https://github.com/kljunowsky/CVE-2022-44268)を使えそうなのでこれを使うと、`464c41477b3768655f736563306e645f663161395f31735f77343174316e395f6630725f793075217d0a`が出てくるのでCyberChefでFrom Hexに渡せばフラグを取得可能。
