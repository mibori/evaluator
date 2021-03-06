[![Build Status](https://travis-ci.org/nullne/evaluator.svg?branch=master)](https://travis-ci.org/nullne/evaluator.svg?branch=master)
[![Coverage Status](https://coveralls.io/repos/github/nullne/evaluator/badge.svg)](https://coveralls.io/github/nullne/evaluator)
[![GoDoc](https://godoc.org/github.com/nullne/evaluator?status.svg)](http://godoc.org/github.com/nullne/evaluator)
[![Go Report Card](https://goreportcard.com/badge/github.com/nullne/evaluator)](https://goreportcard.com/report/github.com/nullne/evaluator)

It's very common to evaluate an expression dynamicly, so that's why we are here.  
#### S-expression
We use s-epxression syntax to parse and evaluate.
> In computing, s-expressions, sexprs or sexps (for "symbolic expression") are a notation for nested list (tree-structured) data, invented for and popularized by the programming language Lisp, which uses them for source code as well as data. 

For example, for expression in common way:
​	
	(
			(gender = "female")
	 and  
	 		((age % 2) != 0)
	 )
it's coresponding format in s-expression is:
​	
	(and
		(= gender "female")
		(!= 
			(% age 2)
			0
		)
	)

#### Element types within expression
- number  
  For convenience, we treat float64, int64 and so on as type of number. For example, float `100.0` is equal to int `100`, but not euqal to string `"100"`

- string  
   character string quoted with `` ` ``, `'`, or `"` are treated as type of `string`. You can convert type `string` to any other defined type you like by type convert functions which are mentioned later

- function or variable  
    character string without quotes are regarded as type of `function` or `variable` which depends on whether this function exists. For example in expression `(age birthdate)`, both `age` and `birthdate` is unquoted. `age` is type of function because we have registered a function named `age`, while `birthdate` is type of variable for not found. The program will come to errors if there is neither parameter nor function named `birthdate` when evaluating


#### How to
You can evaluate directly:

    params := evaluator.MapParams{
        "gender": "female",
    }
    res, err := evaluator.EvalBool(`(in gender ("female" "male"))`, params)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(res)
    # true	

or you can reuse the `Expression` to evaluate multiple times:

    params := evaluator.MapParams{
        "gender": "female",
    }
    exp, err := evaluator.New(`(in gender ("female" "male"))`)
    if err != nil {
        log.Fatal(err)
    }
    res, err := exp.EvalBool(params)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(res)
    # true

##### And you can write expressions like this
- `(in gender ("male", "female"))`
- `(between now (td_time "2017-01-02 12:00:00") (td_time "2017-12-02 12:00:00"))`
- `(ne (mod (age birthdate) 7) 5)`
- or multiple-line for clarity

  ```
  (and
  	(ne os "ios")
  	(eq gender "male")
  	(beteen version (t_version "2.7.1") (t_version "2.9.1"))
  )
  ```


#### Functions
##### Implemented functions


| operand | function  | example | description 
| ------- | --------- | ---------- | ----
| -       | `in`      | `(in 1 (1 2))` | also suport array like `(in (1) ((1)))`
| -       | `between` | `(between age 18 20)` 
| -       | `overlap` | `(overlap region (3142 1860))` 
| `&`     | `and`     | `(and (eq gender "femal") (between age 18 20))`
| `|`     | `or`      |
| `!`     | `not`     |
| `=`     | `eq`      | | equal
| `!=`    | `ne`      | | not equal
| `>`     | `gt`      | | greater than
| `<`     | `lt`      | | less than
| `>=`    | `ge`      | | greater than or equal to
| `<=`    | `le`      | | less than or equal to
| `%`     | `mod`     |
| `+`     | -         | | plus 
| `-`     | -         | | minus
| `*`     | -         | | multiply
| `/`     | -         | | divide
| -       | `t_version` ||  convert type to version
| -       | `t_time`    |`(t_time "2006-01-02 15:04" "2017-09-09 12:00")`| convert type to time, first param must be the layout for the time
| -       | `td_time` |`(td_time "2017:09:09 12:00:00)`| convert type to time of default layout format `2006-01-02 15:04:05`
| _       | `td_date`    |`(in (td_date now) (td_date ("2017-01-02" "2017-02-01")) )`| convert type to time  of default layout format `2006-01-02`

p.s. either operand or function can be used in expression
##### How to use self-defined functions
Yes, you can write your own function by following thses steps:

1. implement your function
2. regist to functions
3. enjoy it

here is an example:

```go
package main


import (
	"errors"
	"log"
	"time"

	"github.com/nullne/evaluator"
	"github.com/nullne/evaluator/function"
)

// define your own function and don't forget to register
func age(params ...interface{}) (interface{}, error) {
	if len(params) != 1 {
		return nil, errors.New("only one params accepted")
	}
	birth, ok := params[0].(string)
	if !ok {
		return nil, errors.New("birth format need to be string")
	}
	r, err := time.Parse("2006-01-02", birth)
	if err != nil {
		return nil, err
	}
	now := time.Now()
	a := r.Year() - now.Year()
	if r.Month() < now.Month() {
		a--
	} else if r.Month() == now.Month() {
		if r.Day() < now.Day() {
			a--
		}
	}
	return a, nil
}

func main() {
	if err := function.Regist("age", age); err != nil {
		log.Print(err)
	}

	exp := `(not (between (age birthdate) 18 20))`
	vvf := evaluator.MapParams{
		"birthdate": "1980-02-01",
	}
	e, err := evaluator.New(exp)
	if err != nil {
		log.Print(err)
	}
	r, err := e.Eval(vvf)
	if err != nil {
		log.Print(err)
	}
	log.Printf("expression: `%s`, wanna: %+v, got: %+v\r", exp, true, r)
}
```


#### Params
- `Params` interface, which has a method named `Get` to get all params needed
- `MapParams` a simple implemented `Params` in `map`

#### Bench
    BenchmarkEqualString-8   	 3000000	       473 ns/op
    BenchmarkInString-8      	 2000000	       916 ns/op
    BenchmarkBetweenInt-8    	 3000000	       467 ns/op
    BenchmarkBetweenTime-8   	 1000000	      2089 ns/op
    BenchmarkOverlapInt-8    	  500000	      2966 ns/op
    BenchmarkTypeTime-8      	 2000000	       638 ns/op
    BenchmarkTypeVersion-8   	 3000000	       539 ns/op

p.s. on MacBook Pro (Retina, 15-inch, Mid 2015), Memory: 16 GB 1600 MHz DDR3, Processor: 2.2 GHz Intel Core i7
