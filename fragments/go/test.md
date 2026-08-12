# Go tests

These rules apply to new or changed tests. Existing tests that break them are
a sidetrack, not part of the change.

## Structure

- **One top-level `TestFoo` per exported symbol.** Each behaviour of `Foo` is
  a `t.Run` subtest.
- **A subtest name states the expected behaviour,** briefly:
  `"rejects empty name"`, `"returns cached value on second call"`. Never
  `"case 1"` or `"valid input"`.
- **Use a table when cases share structure** and differ only in inputs and
  expected outputs. Each row is one partition or boundary, and its name says
  which. Cases that need different setup or assertions get their own subtest.

## Parallelism

Call `t.Parallel()` as the first statement of every test and every subtest.
So a test can't touch process-wide state: no `t.Setenv`, no `t.Chdir`, no
mutable package-level state. Dependencies are passed in, which makes this
possible.

## Assertions

Use `testify`.
- **`require`** stops the test. Use it when later assertions depend on the
  result, such as a nil error before using the value.
- **`assert`** records the failure and continues. Use it for independent
  checks, so one run reports every broken property.

```go
func TestParse(t *testing.T) {
	t.Parallel()

	tests := []struct {
		name    string
		in      string
		want    Config
		wantErr error
	}{
		{name: "parses minimal config", in: "a=1", want: Config{A: 1}},
		{name: "rejects empty input", in: "", wantErr: ErrEmpty},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()

			got, err := Parse(tt.in)
			if tt.wantErr != nil {
				require.ErrorIs(t, err, tt.wantErr)
				return
			}
			require.NoError(t, err)
			assert.Equal(t, tt.want, got)
		})
	}
}
```

## Review signals

- several exported symbols tested in one `TestX`, or one symbol split across
  top-level tests;
- a missing `t.Parallel()`, or a test that needs to run serially;
- subtest or row names that don't state a behaviour;
- table rows that cover the same partition;
- `assert` where a failure makes later lines panic or mislead, or `require`
  on independent checks.
