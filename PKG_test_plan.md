# PKG Test design

- Goal
  - ensure the functionality works correctly
  - increase test coverage of pkg package
  - refactor implementation code

- What to test
  - Priority: Agent -> LB -> Discoverer -> Index manager -> Sidecar -> Filter
  - For now implement pkg/handler only
    - pkg/service will work after it

- How to proceed
  1. test case design share
  2. implementation (refactor impl. if needed)
  3. Check test coverage to avoid missing branchs
  4. code review

# Test methodlogy

- White box testing
  - Statement Coverage
    - all the executable statements in the source code are executed at least once
  - Decision Coverage
    - checking and ensuring that each branch (true / false) of every possible decision point is executed at least once
  - Branch Coverage
    - each decision condition from every branch is executed at least once
  - Condition Coverage
    - check individual outcomes for each logical condition

- Black box testing
  - Equivalence Class Testing
    - divide the test data into the group and then has a representative for each group (https://www.quora.com/What-is-a-real-life-example-for-equivalence-class-testing?share=1)
  - Boundary Value Testing
    - Boundary value testing is focused on the values at boundaries. This technique determines whether a certain range of values are acceptable by the system or not. 
  - Decision Table Testing
    - A decision table puts causes and their effects in a matrix. There is a unique combination in each column.

- Use mock for external dependencies (3rd party dependencies)
  - why: to separate the testing target, reduce dependencies
  - we may need to refactor the implementation to use mock when needed
  - no mock for internal dependencies (may have exception)

# Test cases

To apply white box & black box testing methodlogies above, the key point is:

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

In this case, the mind is to testing different input vector with different settings, which testing the boundary values.

For example we should include the cases that the insert vector is a normal value, empty, 0 value, max value, duplicated and etc.

For different settings, it contains the SkipStrictExistCheck config, which we should test the cases with this flag on/off.

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

## Example 2

StreamInsert [Code](https://github.com/vdaas/vald/blob/master/pkg/agent/core/ngt/handler/grpc/handler.go#L1175)

Since logic is hidden in BidirectionalStream call, we may need to refactor to separate the logic from line 184 to other function to test this function.
https://github.com/vdaas/vald/blob/master/pkg/agent/core/ngt/handler/grpc/handler.go#L1184

This function accept the `Insert_StreamInsertServer` to create the BidirectionalStream. 

The implementation of `Insert_StreamInsertServer` is generated. We can mock the insert server to simplify the test.

The test cases are similar with `insert` function, except we need to handle multiple insert cases:

- Insert multiple vectors success
- Insert multiple vectors with same UUID
- Insert multiple vectors success but one failed
  - we need to check if all vectors sucessfully inserted are inserted

The function return error so we only need to validate the return value is excepted.
