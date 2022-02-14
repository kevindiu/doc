# insert handlerのtest

## White box or black boxどっちやるの?

Black box testing
  - it implements business logic so we use black box

# test case design

The function accept [insert request object](https://github.com/vdaas/vald/blob/master/apis/grpc/v1/payload/payload.pb.go#L1202), which includes the object_vector and the config.

To create boundary values, we need to thing about

- empty value (for slice)
- 0 value
- nil value
- min value of the variable type (if any)
- max value of the variable type (if any)
- Unexcepted/undefined value (e.g. NaN offloating point number)

### test cases

- Equivalence Class Testing
  - case 1: Insert int vector success
  - case 2: Insert float vector success
  - case 3: Insert vector with different dimension fail

- Boundary Value Testing
  - case 1: Insert vector with 0 value success (0 value vector)
  - case 2: Insert vector with min value success (min value vector)
  - case 3: Insert vector with max value success (max value vector)
  - case 4: Insert vector with nil value fail (nil value vector)
  - case 5: Insert empty vector fail (empty vector)
  - case 6: Insert with empty UUID fail (empty UUID)

- Decision Table Testing
  - case 1: Insert duplicated vector success when SkipStrictExistCheck is off
  - case 2: Insert duplicated vector fail when SkipStrictExistCheck is on
