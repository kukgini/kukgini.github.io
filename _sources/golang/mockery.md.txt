# mockery

mockery 는 golang interface 로 부터 mock 의 생성을 자동화 한 도구이다. 이것을 통해 mock 기반의 테스트를 위해 불필요한 상용구 (boilerplate code) 를 작성하지 않아도 된다. 어떻게 쓰는 것인지 살펴보자.

## Installation

```
go get github.com/vektra/mockery/.../
```

그럼 다음 위치에 실행파일이 만들어진다.

```
$GOPATH/bin/mockery
```

## Examples

### Simplest case

`string.go` 가 다음과 같다고 가정해보자.

```go
package test

type Stringer interface {
	String() string
}
```

`mockery --name=Stringer` 를 실행하면 다음 파일이 떨어진다 `mocks/Stringer.go`

```
package mocks

import "github.com/stretchr/testify/mock"

type Stringer struct {
	mock.Mock
}

func (m *Stringer) String() string {
	ret := m.Called()

	var r0 string
	if rf, ok := ret.Get(0).(func() string); ok {
		r0 = rf()
	} else {
		r0 = ret.Get(0).(string)
	}

	return r0
}
```

### Next Level

AWS S3 API 를 테스트하는 코드를 작성해보자. 파일명은 `aws-s3-mock-test.go` 로 하자.

```
package main

import (
	"fmt"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/jaytaylor/mockery-example/mocks"
	"github.com/stretchr/testify/mock"
)

func main() {
	mockS3 := &mocks.S3API{}

	mockResultFn := func(input *s3.ListObjectsInput) *s3.ListObjectsOutput {
		output := &s3.ListObjectsOutput{}
		output.SetCommonPrefixes([]*s3.CommonPrefix{
			&s3.CommonPrefix{
				Prefix: aws.String("2017-01-01"),
			},
		})
		return output
	}

	// NB: .Return(...) must return the same signature as the method being mocked.
	//     In this case it's (*s3.ListObjectsOutput, error).
	mockS3.On("ListObjects", mock.MatchedBy(func(input *s3.ListObjectsInput) bool {
		return input.Delimiter != nil && *input.Delimiter == "/" && input.Prefix == nil
	})).Return(mockResultFn, nil)

	listingInput := &s3.ListObjectsInput{
		Bucket:    aws.String("foo"),
		Delimiter: aws.String("/"),
	}
	listingOutput, err := mockS3.ListObjects(listingInput)
	if err != nil {
		panic(err)
	}

	for _, x := range listingOutput.CommonPrefixes {
		fmt.Printf("common prefix: %+v\n", *x)
	}

```

실행해보자. 다음과 같은 결과를 확인할 수 있을것이다.

```
$ go run aws-s3-mock-test.go
common prefix: {
  Prefix: "2017-01-01"
}
```