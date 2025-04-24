# `protoc-gen-embedjson`

A plugin for `protoc` that generates `MarshalJSON` and `UnmarshalJSON` methods for protobuf messages with fields marked as `embed = true` via `go.field`.

When using `(go.field).embed = true` together with [alta/protopatch](https://github.com/alta/protopatch), a field is generated as a pointer with a JSON tag.  
By default, such fields are serialized as nested objects.  
`protoc-gen-embedjson` solves this by allowing these structures to be (de)serialized as flat objects.

---

## Features

- Generates `_json.pb.go` files with `MarshalJSON` and `UnmarshalJSON` methods **only for messages that contain embedded fields**.
- Automatically removes `_json.pb.go` files when embedded fields are removed from `.proto` definitions.

---

## Installation

```sh
cd protoc-gen-embedjson
go build -o bin/protoc-gen-embedjson
```

---

## Usage

```sh
protoc -I=. \
  -I=$GOPATH/src \
  -I=$GOPATH/src/github.com/alta/protopatch/patch \
  --plugin=protoc-gen-embedjson=../../protoc-gen-embedjson/bin/protoc-gen-embedjson \
  --embedjson_out=. ${1:-*.proto}
```

---

## Example

**Sample `.proto` file with a shared message: `common.task.proto`**

```proto
syntax = "proto3";
import "github.com/alta/protopatch/patch/go.proto";
option go_package = "./models";
package common.task;

message Common_Task_Response {
  int64 id = 1 [(go.field).tags = 'json:"id" validate:"required"'];
  string name = 2 [(go.field).tags = 'json:"name" validate:"required"'];
}
```

**Sample `.proto` file using the shared message: `task.get.proto`**

```proto
syntax = "proto3";
import "github.com/alta/protopatch/patch/go.proto";
import "common.task.proto";
option go_package = "./models";

message TaskGet {
  message Response {
    common.task.Common_Task_Response embedded = 1 [(go.field).embed = true];
  }
}
```

**Generated Go structure in `task.get.pb.go` will look like this:**

```go
type TaskGet_Response struct {
	*Common_Task_Response `json:"embedded,omitempty"`
}
```

**Generated `_json.pb.go` file `task.get_json.pb.go`:**

```go
package models

import "encoding/json"

func (r *TaskGet_Response) MarshalJSON() ([]byte, error) {
	if r.Common_Task_Response == nil {
		return json.Marshal(struct{}{})
	}
	return json.Marshal(r.Common_Task_Response)
}

func (r *TaskGet_Response) UnmarshalJSON(data []byte) error {
	if r.Common_Task_Response == nil {
		r.Common_Task_Response = &Common_Task_Response{}
	}
	return json.Unmarshal(data, r.Common_Task_Response)
}
```

Now you can work with the object as a flat structure:

```go
var response models.TaskGet_Response
_ = json.Unmarshal([]byte(`{"name": "test"}`), &response)
fmt.Println(response.Name)
// Output: test
// Without custom UnmarshalJSON: panic `invalid memory address or nil pointer dereference`

bytes, _ := json.Marshal(&response)
fmt.Println(string(bytes))
// Output: {"id":0,"name":"test"}
// Without custom MarshalJSON: {}
```

---

## Requirements

- `protoc` >= 3.15
- Go >= 1.18
- [`github.com/alta/protopatch`](https://github.com/alta/protopatch)
