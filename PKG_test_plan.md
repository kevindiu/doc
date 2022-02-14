# PKG Test design

# Test cases

To apply grey box testing methodlogies, the key point is:

- cover valid and invalid case
- try to cover most of the conditions (if-else statement)
  - for test coverage and also ensure every code execute at least once
- try to cover all of the executable statement
- test boundary cases
- test with different input with different settings

## Example 1

We take agent/handler insert function as example. [Code](https://github.com/vdaas/vald/blob/master/pkg/agent/core/ngt/handler/grpc/handler.go#L1079)

### test cases

The function accept `ctx context.Context` and `req *payload.Insert_Request`, which contains the insert vector and configuration.

In this case, the mind is to test different input vectors with different settings, and test the boundary values.

For example we should include the cases that the insert vector is: 

- normal value
- empty vector
- 0 value vector
- max value vector
- duplicated vector

For different settings, it contains the SkipStrictExistCheck config, which we should test the cases with this flag on/off.

- SkipStrictExistCheck on with duplicated vector
- SkipStrictExistCheck off with duplicated vector
- SkipStrictExistCheck on with non-duplicated vector
- SkipStrictExistCheck off with non-duplicated vector

We can see the source code and to cover white box cases like below comments in the source code.

```go
func (s *server) Insert(ctx context.Context, req *payload.Insert_Request) (res *payload.Object_Location, err error) {
	_, span := trace.StartSpan(ctx, apiName+".Insert")
	defer func() {
		if span != nil { // branch coverage case 5
			span.End()
		}
	}()
	vec := req.GetVector()
	if len(vec.GetVector()) != s.ngt.GetDimensionSize() { // branch coverage case 1
		err = errors.ErrIncompatibleDimensionSize(len(vec.GetVector()), int(s.ngt.GetDimensionSize()))
		err = status.WrapWithInvalidArgument("Insert API Incompatible Dimension Size detected",
			...
		log.Warn(err)
		if span != nil {
			span.SetStatus(trace.StatusCodeInvalidArgument(err.Error()))
		}
		return nil, err
	}

	err = s.ngt.InsertWithTime(vec.GetId(), vec.GetVector(), req.GetConfig().GetTimestamp()) // boundary value testing, Equivalence Class Testing
	if err != nil {
		var code trace.Status

		if errors.Is(err, errors.ErrUUIDAlreadyExists(vec.GetId())) { // branch coverage case 3
			err = status.WrapWithAlreadyExists(fmt.Sprintf("Insert API uuid %s already exists", vec.GetId()), err,
			...
			log.Warn(err)
			code = trace.StatusCodeAlreadyExists(err.Error())
		} else if errors.Is(err, errors.ErrUUIDNotFound(0)) { // boundary value testing case 3 (can be branch coverage)
			err = status.WrapWithInvalidArgument(fmt.Sprintf("Insert API empty uuid \"%s\" was given", vec.GetId()), err,
			...
			log.Warn(err)
			code = trace.StatusCodeInvalidArgument(err.Error())
		} else { // branch coverage case 4
			var (
				st  *status.Status
				msg string
			)
			st, msg, err = status.ParseError(err, codes.Internal,
			...
			code = trace.FromGRPCStatus(st.Code(), msg)
		}
		if span != nil { // branch coverage case 5
			span.SetStatus(code)
		}
		return nil, err
	}
	return s.newLocation(vec.GetId()), nil
}
```

Example test case:

- Black box testing
  - Equivalence Class Testing
    - case 1: Insert vector success
  - Boundary Value Testing
    - case 1: Insert vector with 0 value success
    - case 2: Insert vector with max value success
    - case 3: Insert with empty UUID fail
    - case 4: Insert empty vector fail (0 length vector)
  
- White box testing
  - Branch Coverage
    - case 1: Insert vector with different dimension fail
    - case 2: Insert duplicated vector success when SkipStrictExistCheck is off
    - case 3: Insert duplicated vector fail when SkipStrictExistCheck is on
    - case 4: InsertWithTime return other error
    - case 5: Trace span is not nil

Since there are no 3rd party dependencies, no mock will be applied.

### Validate output

Since the function return Object_Location and error, we will validate if the returned location and error is excepted.
