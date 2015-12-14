[*back to contents*](https://github.com/gyuho/learn#contents)<br>

# Go: boltdb

- [Reference](#reference)
- [Generate random data](#generate-random-data)
- [Read all data](#read-all-data)

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Reference

- [`boltdb/bolt`](https://github.com/boltdb/bolt)

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Generate random data

```go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"os/user"
	"path/filepath"
	"time"

	"github.com/boltdb/bolt"
)

const (
	// these will create 2GB database.
	numKeys = 500000
	keyLen  = 100
	valLen  = 750

	bucketName = "test_bucket"
	writable   = true
)

func main() {
	usr, err := user.Current()
	if err != nil {
		panic(err)
	}
	dbPath := filepath.Join(usr.HomeDir, "test.db")

	if err := os.RemoveAll(dbPath); err != nil {
		panic(err)
	}

	// Open the dbPath data file in your current directory.
	// It will be created if it doesn't exist.
	db, err := bolt.Open(dbPath, 0600, &bolt.Options{Timeout: 1 * time.Second})
	if err != nil {
		panic(err)
	}
	defer db.Close()

	// Start a writable transaction.
	tx, err := db.Begin(writable)
	if err != nil {
		panic(err)
	}
	defer tx.Rollback()

	// Use the transaction
	b, err := tx.CreateBucket([]byte(bucketName))
	if err != nil {
		panic(err)
	}

	// Write to database
	for i := 0; i < numKeys; i++ {
		k := randBytes(keyLen)
		v := randBytes(valLen)
		fmt.Println("Writing", i, "/", numKeys)
		if err := b.Put(k, v); err != nil {
			panic(err)
		}
	}

	// Commit the transaction
	fmt.Println("Committing...")
	if err := tx.Commit(); err != nil {
		panic(err)
	}

	fmt.Println("Done")
}

const (
	// http://stackoverflow.com/questions/22892120/how-to-generate-a-random-string-of-a-fixed-length-in-golang
	letterBytes   = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
	letterIdxBits = 6                    // 6 bits to represent a letter index
	letterIdxMask = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
	letterIdxMax  = 63 / letterIdxBits   // # of letter indices fitting in 63 bits
)

func randBytes(n int) []byte {
	src := rand.NewSource(time.Now().UnixNano())
	b := make([]byte, n)
	// A src.Int63() generates 63 random bits, enough for letterIdxMax characters!
	for i, cache, remain := n-1, src.Int63(), letterIdxMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdxMax
		}
		if idx := int(cache & letterIdxMask); idx < len(letterBytes) {
			b[i] = letterBytes[idx]
			i--
		}
		cache >>= letterIdxBits
		remain--
	}
	return b
}

```

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Read all data

```go
package main

import (
	"fmt"
	"os/user"
	"path/filepath"
	"syscall"
	"time"

	"github.com/boltdb/bolt"
)

const (
	bucketName = "test_bucket"
	writable   = false
)

func main() {
	fmt.Println("read")
	read()

	fmt.Println()
	fmt.Println("readMapPopulate")
	readMapPopulate()
}

/*
read
bolt.Open took 47.419µs
bolt Read took: 75.532225ms

readMapPopulate
bolt.Open took 254.063181ms
bolt Read took: 51.274025ms
*/

func read() {
	usr, err := user.Current()
	if err != nil {
		panic(err)
	}
	dbPath := filepath.Join(usr.HomeDir, "test.db")

	to := time.Now()
	db, err := bolt.Open(dbPath, 0600, &bolt.Options{Timeout: 5 * time.Minute, ReadOnly: true})
	if err != nil {
		panic(err)
	}
	defer db.Close()
	fmt.Println("bolt.Open took", time.Since(to))

	tr := time.Now()
	tx, err := db.Begin(writable)
	if err != nil {
		panic(err)
	}
	defer tx.Rollback()

	bk := tx.Bucket([]byte(bucketName))
	c := bk.Cursor()

	for k, v := c.First(); k != nil; k, v = c.Next() {
		// fmt.Printf("%s ---> %s.\n", k, v)
		_ = k
		_ = v
	}
	fmt.Println("bolt Read took:", time.Since(tr))
}

func readMapPopulate() {
	usr, err := user.Current()
	if err != nil {
		panic(err)
	}
	dbPath := filepath.Join(usr.HomeDir, "test.db")

	to := time.Now()
	db, err := bolt.Open(dbPath, 0600, &bolt.Options{Timeout: 5 * time.Minute, ReadOnly: true, MmapFlags: syscall.MAP_POPULATE})
	if err != nil {
		panic(err)
	}
	defer db.Close()
	fmt.Println("bolt.Open took", time.Since(to))

	tr := time.Now()
	tx, err := db.Begin(writable)
	if err != nil {
		panic(err)
	}
	defer tx.Rollback()

	bk := tx.Bucket([]byte(bucketName))
	c := bk.Cursor()

	for k, v := c.First(); k != nil; k, v = c.Next() {
		// fmt.Printf("%s ---> %s.\n", k, v)
		_ = k
		_ = v
	}
	fmt.Println("bolt Read took:", time.Since(tr))
}

```

[↑ top](#go-boltdb)
<br><br><br><br><hr>
