The following new Condition types are added:
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
