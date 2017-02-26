# Venom

Venom run executors (script, HTTP Request, etc... ) and assertions.
It can also output xUnit results files.

<img src="./venom.gif" alt="Venom Demonstration" width="80%">

## Commmand Line

Install with:
```bash
$ go get github.com/runabove/venom
$ go install github.com/runabove/venom/cli/venom
```

```bash
$ venom run -h
Run Tests

Usage:
  venom run [flags]

Flags:
      --alias stringSlice   --alias cds:'cds -f config.json' --alias cds2:'cds -f config.json'
      --details string      Output Details Level : low, medium, high (default "medium")
      --format string       --formt:yaml, json, xml (default "xml")
      --log string          Log Level : debug, info or warn (default "warn")
      --output-dir string   Output Directory: create tests results file inside this directory
      --parallel int        --parallel=2 (default 1)
      --resume              Output Resume: one line with Total, TotalOK, TotalKO, TotalSkipped, TotalTestSuite (default true)
      --resumeFailures      Output Resume Failures (default true)
```

## TestSuite files

* Run `venom template`
* Examples: https://github.com/ovh/cds/tree/master/tests

### Example:

```yaml

name: Title of TestSuite
testcases:
- name: TestCase with default value, exec cmd. Check if exit code != 1
  steps:
  - script: echo 'foo'
    type: exec

- name: Title of First TestCase
  steps:
  - script: echo 'foo'
    assertions:
    - result.code ShouldEqual 0
  - script: echo 'bar'
    assertions:
    - result.systemout ShouldNotContainSubstring foo
    - result.timeseconds ShouldBeLessThan 1

- name: GET http testcase, with 5 seconds timeout
  steps:
  - type: http
    method: GET
    url: https://eu.api.ovh.com/1.0/
    timeout: 5
    assertions:
    - result.body ShouldContainSubstring /dedicated/server
    - result.body ShouldContainSubstring /ipLoadbalancing
    - result.statuscode ShouldEqual 200
    - result.timeseconds ShouldBeLessThan 1

- name: Test with retries and delay in seconds between each try
  steps:
  - type: http
    method: GET
    url: https://eu.api.ovh.com/1.0/
    retry: 3
    delay: 2
    assertions:
    - result.statuscode ShouldEqual 200

```

Using variables and reuse results

```yaml
name: MyTestSuiteTmpl
vars:
  api.foo: 'http://api/foo'
  second: 'venomWithTmpl'

testcases:
- name: testA
  steps:
  - type: exec
    script: echo '{{.api.foo}}'
    assertions:
    - result.code ShouldEqual 0
    - result.systemout ShouldEqual http://api/foo

- name: testB
  steps:
  - type: exec
    script: echo 'XXX{{.testA.result.systemout}}YYY'
    assertions:
    - result.code ShouldEqual 0
    - result.systemout ShouldEqual XXXhttp://api/fooYYY

```

## RUN Venom locally on CDS Integration Tests

```bash
cd $GOPATH/src/github.com/ovh/cds/tests
venom run --alias='cdsro:cds -f $HOME/.cds/it.user.ro.json' --alias='cds:cds -f $HOME/.cds/it.user.rw.json' --parallel=5
```

## RUN Venom, with an export xUnit

```bash
venom run  --details=low --format=xml --output-dir="."
```

## Assertion

### Keywords
* ShouldEqual
* ShouldNotEqual
* ShouldAlmostEqual
* ShouldNotAlmostEqual
* ShouldResemble
* ShouldNotResemble
* ShouldPointTo
* ShouldNotPointTo
* ShouldBeNil
* ShouldNotBeNil
* ShouldBeTrue
* ShouldBeFalse
* ShouldBeZeroValue
* ShouldBeGreaterThan
* ShouldBeGreaterThanOrEqualTo
* ShouldBeLessThan
* ShouldBeLessThanOrEqualTo
* ShouldBeBetween
* ShouldNotBeBetween
* ShouldBeBetweenOrEqual
* ShouldNotBeBetweenOrEqual
* ShouldContain
* ShouldNotContain
* ShouldContainKey
* ShouldNotContainKey
* ShouldBeIn
* ShouldNotBeIn
* ShouldBeEmpty
* ShouldNotBeEmpty
* ShouldHaveLength
* ShouldStartWith
* ShouldNotStartWith
* ShouldEndWith
* ShouldNotEndWith
* ShouldBeBlank
* ShouldNotBeBlank
* ShouldContainSubstring
* ShouldNotContainSubstring
* ShouldEqualWithout
* ShouldEqualTrimSpace
* ShouldHappenBefore
* ShouldHappenOnOrBefore
* ShouldHappenAfter
* ShouldHappenOnOrAfter
* ShouldHappenBetween
* ShouldHappenOnOrBetween
* ShouldNotHappenOnOrBetween
* ShouldHappenWithin
* ShouldNotHappenWithin
* ShouldBeChronological

## Executors

### Exec

`exec` is the default type for a step

See https://github.com/runabove/venom/tree/master/executors/exec

### HTTP

See https://github.com/runabove/venom/tree/master/executors/http

### Read file

See https://github.com/runabove/venom/tree/master/executors/readfile

### SMTP

See https://github.com/runabove/venom/tree/master/executors/smtp

### SSH

See https://github.com/runabove/venom/tree/master/executors/ssh

### WEB

See https://github.com/runabove/venom/tree/master/executors/web

### Write your executor

An executor have to implement this interface

```go

// Executor execute a testStep.
type Executor interface {
	// Run run a Test Step
	Run(ctx context.Content, *log.Entry, Aliases, TestStep) (ExecutorResult, error)
}
```

Example

```go


// Name of executor
const Name = "myexecutor"

// New returns a new Executor
func New() venom.Executor {
	return &Executor{}
}

// Executor struct
type Executor struct {
	Command string `json:"command,omitempty" yaml:"command,omitempty"`
}

// Result represents a step result
type Result struct {
	Code        int    `json:"code,omitempty" yaml:"code,omitempty"`
	Command     string `json:"command,omitempty" yaml:"command,omitempty"`
	Systemout   string   `json:"systemout,omitempty" yaml:"systemout,omitempty"` // put in testcase.Systemout by venom if present
	Systemerr   string   `json:"systemerr,omitempty" yaml:"systemerr,omitempty"` // put in testcase.Systemerr by venom if present
	Executor    Executor `json:"executor,omitempty" yaml:"executor,omitempty"`  
}

// GetDefaultAssertions return default assertions for this executor
// Optional
func (Executor) GetDefaultAssertions() venom.StepAssertions {
	return venom.StepAssertions{Assertions: []string{"result.code ShouldEqual 0"}}
}

// Run execute TestStep
func (Executor) Run(ctx context.Context, l *log.Entry, aliases venom.Aliases, step venom.TestStep) (venom.ExecutorResult, error) {

	// transform step to Executor Instance
	var t Executor
	if err := mapstructure.Decode(step, &t); err != nil {
		return nil, err
	}

	// Get testcase context if needed
	varContext := ctx.Value(venom.ContextKey).(map[string]interface{})
	if varContext == nil {
        return nil, fmt.Errorf("Executor web need a context")
    }
    bar := varContext['foo']

	// to something with t.Command here...
	//...

	systemout := "foo"
	ouputCode := 0

	// prepare result
	r := Result{
		Code:    ouputCode, // return Output Code
		Command: t.Command, // return Command executed
		Systemout:  systemout,    // return Output string
		Executor: t, // return executor, usefull for display Executor context in failure
	}

	return dump.ToMap(r)
}

```

Feel free to open a Pull Request with your executors.


## TestCase Context

TestCase Context allows you to inject datas in all Steps.

Define a context is optional, but can be usefull to keep data between teststeps on a testcase.

### Write your TestCase Context

A TestCase Context has to implement this interface

```go

type TestCaseContext interface {
	Init() error
	Close() error
	SetTestCase(tc TestCase)
	GetName() string
}
```

Example

```go
// Context Type name
const Name = "default"

// New returns a new TestCaseContext
func New() venom.TestCaseContext {
	ctx := &DefaultTestCaseContext{}
	ctx.Name = Name
	return ctx
}

// DefaultTestCaseContext represents the context of a testcase
type DefaultTestCaseContext struct {
	venom.CommonTestCaseContext
	datas map[string]interface{}
}

// Init Initialize the context
func (tcc *DefaultTestCaseContext) Init() error {
	return nil
}

// Close the context
func (tcc *DefaultTestCaseContext) Close() error {
	return nil
}
```
Methods SetTestCase and  GetName are implemented by CommonTestCaseContext

# Hacking

You've developed a new cool feature? Fixed an annoying bug? We'd be happy
to hear from you! Make sure to read [CONTRIBUTING.md](./CONTRIBUTING.md) before.

# License

This work is under the BSD license, see the [LICENSE](LICENSE) file for details.
