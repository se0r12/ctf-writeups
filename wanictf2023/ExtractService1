## Extract Service 1

基本的な処理は多分以下。

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

		extractTarget := c.PostForm("target")
		if extractTarget == "" {
			c.HTML(http.StatusOK, "index.html", gin.H{
				"result": "Error : target is required",
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

fileパラメータとtargetパラメータを受け取ります。

baseDir (`/tmp/{:uuid}`)をmkdirし、その後zipPath ( {:baseDir}.zip)に対してfileを書き込みます。

次に、`ExtractFile`で、zipPath, baseDirを受取り、`unzip zipPath -d baseDire`をしています。

展開したら、`ExtractContet(baseDir, extractTarget)`で、baseDirにextraTargetを組み込み (つまり `/tmp/{:uuid} + extraTarget`)ファイルの内容を読み取るそうです。

ディレクトリトラバーサル一択っぽいですね。

targetに`../../../../../../../../flag`を指定すればFLAGが見えます。
