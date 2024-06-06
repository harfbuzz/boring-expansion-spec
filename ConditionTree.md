The following new Condition types are added, which enable encoding a tree representing a boolean expression:
```
struct ConditionAnd
{
  uint16 format; // 3
  uint8 conditionCount; // Number of conditions for this conjunction expression.
  Offset24To<Condition> conditionOffsets[conditionCount];
};

struct ConditionOr
{
  uint16 format; // 4
  uint8 conditionCount; // Number of conditions for this disjunction expression.
  Offset24To<Condition> conditionOffsets[conditionCount];
};

struct ConditionNegate
{
  uint16 format; // 5
  Offset24To<Condition> condition;
};
```

While not directly relevant to representing condition trees, the following Condition type is also added:
```
struct ConditionValue
{
  uint16 format; // 2
  int16 defaultValue;
  VarIdx varIdx;
};
```
Here, the condition evaluates to true if and only if `defaultValue` plus the evaluation of `varIdx` in the respective variation-store for current font coordinates is a positive number.
