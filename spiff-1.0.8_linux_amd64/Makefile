GODEPS := $(shell godep path)
GOPATH := $(GODEPS):$(GOPATH)

grammar:
	go get github.com/pointlander/peg
	peg dynaml/dynaml.peg

release: spiff_linux_amd64.zip spiff_darwin_amd64.zip	

spiff_linux_amd64.zip:
	GOPATH=$(GOPATH) GOOS=linux GOARCH=amd64 go build -o spiff .
	zip spiff_linux_amd64.zip spiff
	rm spiff

spiff_darwin_amd64.zip:
	GOPATH=$(GOPATH) GOOS=darwin GOARCH=amd64 go build -o spiff .
	zip spiff_darwin_amd64.zip spiff
	rm spiff
