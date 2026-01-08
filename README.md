```
const selectedClientItem = (itemsOverride ?? touenItems)?.find(item => {
  const classA = item?.ClassHash?.ClassA
  if (classA == null || tokuisakiID == null) return false
  return String(classA).trim() == String(tokuisakiID).trim() // 핵심: trim(+ 필요시 ==)
})

```
