---
title: "HOW TO: Unmarshal multiple yaml documents in Go"
date: 2020-12-10T18:03:51+01:00
draft: false
---

The [README of the go-yaml
package](https://github.com/go-yaml/yaml#compatibility) states:

> The yaml package supports most of YAML 1.1 and 1.2, including 
> support for anchors, tags, map merging, etc. _Multi-document unmarshalling is
> not yet implemented_

Nevertheless, multiple yaml documents can be parsed simply and reliably in Go and
in this small post I'll demonstrate how.

## Breakfast

Let's say I run a surprisingly tech savvy cafe that accepts orders for breakfast
in yaml format. A collection of orders can be provided in the form of multiple
yaml documents:

```golang
var data = `
person: john
items:
  - name: sausage
    quantity: 2
  - name: egg
    quantity: 1
  - name: bacon
    quantity: 2
---
person: tina
items:
  - name: sausage
    quantity: 1
  - name: egg
    quantity: 2
  - name: bacon
    quantity: 2
  - name: toast
    quantity: 1
`
```

Yes, I know this is an odd example.

As with any single yaml document in Go, I can represent it with a type:

```golang
type BreakfastOrder struct {
	Person string `yaml:"person"`
	Items  []struct {
		Name     string `yaml:"name"`
		Quantity int    `yaml:"quantity"`
	} `yaml:"items"`
}
```

If there were only one document I could use `yaml.Unmarshal` to unmarshal the
yaml data into an instance of this type. However, because `yaml.Unmarshal`
doesn't support multiple documents, you need to write a bit more code.

Here's a function that will iterate over multiple documents and decode them into a
`[]BreakfastOrder` slice:

```golang
func UnmarshalAllBreakfastOrders(in []byte, out *[]BreakfastOrder) error {
	r := bytes.NewReader(in)
	decoder := yaml.NewDecoder(r)
	for {
		var bo BreakfastOrder
		if err := decoder.Decode(&bo); err != nil {
			// Break when there are no more documents to decode
			if err != io.EOF {
				return err
			}
			break
		}
		*out = append(*out, bo)
	}
	return nil
}
```

Now I can do this in my main function:

```golang
func main() {
	orders := []BreakfastOrder{}
	if err := UnmarshalAllBreakfastOrders([]byte(data), &orders); err != nil {
		fmt.Println(err)
		return
	}
	for _, order := range orders {
		fmt.Printf("%s wants: %v\n", order.Person, order.Items)
	}
}
```

And get what I want:

```
john wants: [{sausage 2} {egg 1} {bacon 2}]
tina wants: [{sausage 1} {egg 2} {bacon 2} {toast 1}]
```

## Full Example

[Try it on The Go Playground](https://play.golang.org/p/YAavt2Gw_jw)

```
package main

import (
	"bytes"
	"fmt"
	"io"

	"gopkg.in/yaml.v2"
)

var data = `
person: john
items:
  - name: sausage
    quantity: 2
  - name: egg
    quantity: 1
  - name: bacon
    quantity: 2
---
person: tina
items:
  - name: sausage
    quantity: 1
  - name: egg
    quantity: 2
  - name: bacon
    quantity: 2
  - name: toast
    quantity: 1
`

type BreakfastOrder struct {
	Person string `yaml:"person"`
	Items  []struct {
		Name     string `yaml:"name"`
		Quantity int    `yaml:"quantity"`
	} `yaml:"items"`
}

func UnmarshalAllBreakfastOrders(in []byte, out *[]BreakfastOrder) error {
	r := bytes.NewReader(in)
	decoder := yaml.NewDecoder(r)
	for {
		var bo BreakfastOrder

		if err := decoder.Decode(&bo); err != nil {
			// Break when there are no more documents to decode
			if err != io.EOF {
				return err
			}
			break
		}
		*out = append(*out, bo)
	}
	return nil
}

func main() {
	orders := []BreakfastOrder{}
	if err := UnmarshalAllBreakfastOrders([]byte(data), &orders); err != nil {
		fmt.Println(err)
		return
	}
	for _, order := range orders {
		fmt.Printf("%s wants: %v\n", order.Person, order.Items)
	}
}
```
