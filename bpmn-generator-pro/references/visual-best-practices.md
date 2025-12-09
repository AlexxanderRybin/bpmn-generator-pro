# BPMN Visual Best Practices

Правила для создания читаемых диаграмм без визуальных конфликтов.

## Проблема: Наложение элементов

### Причина 1: Несколько путей к одной точке

**Плохо:**
```
Path A → Task_Success (2270, 110) → End_Success (2442, 132)
Path B → Task_Reject (2270, 110) → End_Reject  (2442, 132)
         ↑ КОНФЛИКТ! Одинаковые координаты
```

**Решение: Разделение по вертикали**
```javascript
// Расчет Y координат для разных путей
const LANE_CENTER = lane.y + lane.height / 2

// Главный путь (success)
success_y = LANE_CENTER

// Альтернативный путь (rejection/error)
alternative_y = LANE_CENTER + 80  // Смещение вниз

// Пример:
Task_Success: (2270, 110)  // center
Task_Reject:  (2270, 190)  // center + 80

End_Success:  (2442, 132)
End_Reject:   (2442, 212)  // +80 от success
```

**Правило:**
> Если несколько независимых путей приводят к разным outcomes в одной lane,
> они ДОЛЖНЫ иметь разные Y координаты с минимальным offset 80px

### Причина 2: Пересечение потоков с элементами

**Плохо:**
```
Gateway (940, 445) --[No]--> Task_Notify (2270, 150)
                      ↓
                  Пересекает задачи в Operations lane!
```

**Решение: Routing через коридоры**

#### Концепция "коридоров"

```
Lane 1 (Customer):    y = 50-250
CORRIDOR 1:           y = 250-280  ← 30px свободное пространство
Lane 2 (Operations):  y = 280-720
CORRIDOR 2:           y = 720-750  ← 30px свободное пространство
Lane 3 (Finance):     y = 750-900
```

#### Алгоритм routing

```javascript
function routeFlow(source, target) {
  const sourceLane = findLane(source)
  const targetLane = findLane(target)

  // Если в одной lane - прямая линия
  if (sourceLane === targetLane) {
    return straightLine(source, target)
  }

  // Если нужно пересечь другие lanes
  // 1. Выйти из source lane (вертикально до коридора)
  // 2. Идти по коридору (горизонтально)
  // 3. Войти в target lane (вертикально)

  const corridor = findCorridorBetween(sourceLane, targetLane)

  return [
    exitPoint(source, corridor),        // Выход к коридору
    { x: target.x - 50, y: corridor.y }, // Движение по коридору
    entryPoint(target, corridor)        // Вход в целевую lane
  ]
}
```

**Пример waypoints:**
```xml
<!-- Flow от Gateway (Operations) к Task_Notify (Customer) -->
<bpmndi:BPMNEdge id="Flow_ChecksFailed_di">
  <di:waypoint x="965" y="470"/>          <!-- Выход из gateway -->
  <di:waypoint x="965" y="260"/>          <!-- Вверх к corridor -->
  <di:waypoint x="2220" y="260"/>         <!-- Коридор горизонтально -->
  <di:waypoint x="2220" y="190"/>         <!-- Вниз в Customer lane -->
  <di:waypoint x="2270" y="190"/>         <!-- К задаче -->
</bpmndi:BPMNEdge>
```

## Проблема: Субпроцессы занимают много места

### Решение 1: Collapsed Subprocess

**Вместо:**
```xml
<bpmn:subProcess id="Sub_Payment" isExpanded="true">
  <!-- 590x130px раскрытый subprocess -->
  <!-- Все внутренние элементы видны -->
</bpmn:subProcess>
```

**Используйте:**
```xml
<bpmn:subProcess id="Sub_Payment" isExpanded="false">
  <!-- 100x80px свернутый subprocess -->
  <!-- Детали скрыты, можно раскрыть по клику -->
  <bpmn:startEvent id="SubStart"/>
  <!-- ... внутренние элементы ... -->
  <bpmn:endEvent id="SubEnd"/>
</bpmn:subProcess>

<!-- Визуально отображается как обычная задача с [+] -->
<bpmndi:BPMNShape id="Sub_Payment_di" bpmnElement="Sub_Payment" isExpanded="false">
  <dc:Bounds x="1055" y="725" width="100" height="80"/>
</bpmndi:BPMNShape>
```

### Решение 2: Call Activity

**Для переиспользуемой логики:**
```xml
<!-- Основной процесс -->
<bpmn:callActivity id="Call_Payment" name="Process Payment"
                   calledElement="PaymentProcess">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:callActivity>

<!-- Визуально: 100x80px с толстой рамкой -->
<bpmndi:BPMNShape id="Call_Payment_di" bpmnElement="Call_Payment">
  <dc:Bounds x="1055" y="725" width="100" height="80"/>
</bpmndi:BPMNShape>

<!-- Отдельный файл: payment-process.bpmn -->
<bpmn:process id="PaymentProcess" isExecutable="true">
  <!-- Вся логика платежа в отдельной диаграмме -->
</bpmn:process>
```

## Улучшенные правила компоновки

### 1. Spacing между путями в одной lane

```javascript
const PATH_SEPARATION = {
  MIN_VERTICAL_OFFSET: 80,    // Минимум между альтернативными путями
  PREFERRED_OFFSET: 100,       // Рекомендуемое расстояние
}

// Пример расчета для success/failure paths
function calculateAlternativePathY(mainPath, alternatives) {
  const positions = []

  // Главный путь по центру
  positions.push({
    path: 'main',
    y: lane.centerY
  })

  // Альтернативы со смещением
  alternatives.forEach((alt, index) => {
    positions.push({
      path: alt.name,
      y: lane.centerY + ((index + 1) * PATH_SEPARATION.PREFERRED_OFFSET)
    })
  })

  return positions
}
```

### 2. Corridor spacing

```javascript
const CORRIDOR = {
  HEIGHT: 30,                  // Высота коридора между lanes
  MARGIN_FROM_LANE: 10,        // Отступ от границы lane
}

function calculateCorridors(lanes) {
  const corridors = []

  for (let i = 0; i < lanes.length - 1; i++) {
    const currentLane = lanes[i]
    const nextLane = lanes[i + 1]

    const corridorY = currentLane.y + currentLane.height + CORRIDOR.MARGIN_FROM_LANE

    corridors.push({
      between: [currentLane.id, nextLane.id],
      y: corridorY,
      height: CORRIDOR.HEIGHT
    })
  }

  return corridors
}
```

### 3. Flow routing rules

```javascript
const FLOW_ROUTING = {
  // Минимальное расстояние от элемента при обходе
  CLEARANCE: 20,

  // Предпочтительный тип линий
  PREFER_ORTHOGONAL: true,  // 90° углы, не диагонали

  // Приоритеты routing
  PRIORITY: {
    STRAIGHT: 1,           // Прямая линия (если нет препятствий)
    CORRIDOR: 2,           // Через коридор (если пересекает lanes)
    AROUND: 3,             // Обход элементов (если есть препятствия)
  }
}

function routeFlowIntelligent(source, target, existingElements) {
  const sourceLane = findLane(source)
  const targetLane = findLane(target)

  // 1. Попробовать прямую линию
  const straight = straightLine(source, target)
  if (!hasCollisions(straight, existingElements)) {
    return straight
  }

  // 2. Попробовать через коридор (если разные lanes)
  if (sourceLane !== targetLane) {
    const viaCorrid or = routeViaCorridor(source, target, sourceLane, targetLane)
    if (!hasCollisions(viaCorridor, existingElements)) {
      return viaCorridor
    }
  }

  // 3. Обход препятствий
  return routeAroundObstacles(source, target, existingElements)
}
```

### 4. Subprocess sizing rules

```javascript
const SUBPROCESS_RULES = {
  // По умолчанию создавать свернутыми
  DEFAULT_EXPANDED: false,

  // Порог сложности для авто-сворачивания
  AUTO_COLLAPSE_THRESHOLD: {
    ELEMENTS: 5,          // > 5 элементов → свернуть
    WIDTH: 400,           // > 400px ширина → свернуть
    HEIGHT: 200,          // > 200px высота → свернуть
  },

  // Размеры свернутого subprocess
  COLLAPSED_SIZE: {
    width: 100,
    height: 80
  },

  // Minimum padding для раскрытого
  EXPANDED_PADDING: 30,
}

function shouldCollapseSubprocess(subprocess) {
  const childCount = subprocess.children.length
  const bounds = calculateBounds(subprocess)

  return childCount > SUBPROCESS_RULES.AUTO_COLLAPSE_THRESHOLD.ELEMENTS ||
         bounds.width > SUBPROCESS_RULES.AUTO_COLLAPSE_THRESHOLD.WIDTH ||
         bounds.height > SUBPROCESS_RULES.AUTO_COLLAPSE_THRESHOLD.HEIGHT
}
```

## Пошаговый алгоритм предотвращения конфликтов

### Шаг 1: Группировка путей

```javascript
class ProcessAnalyzer {
  constructor(process) {
    this.process = process
    this.paths = []
  }

  // Определить все возможные пути от start до end
  findAllPaths() {
    const startEvents = this.process.startEvents
    const endEvents = this.process.endEvents

    const paths = []

    for (const start of startEvents) {
      for (const end of endEvents) {
        const path = this.tracePath(start, end)
        if (path) {
          paths.push({
            id: `${start.id}_to_${end.id}`,
            elements: path,
            outcome: end.name,  // "Order Completed", "Order Rejected", etc.
          })
        }
      }
    }

    return paths
  }

  // Группировать пути по lane для одной Y координаты
  groupPathsByLaneAndOutcome() {
    const grouped = {}

    for (const path of this.paths) {
      for (const lane of this.process.lanes) {
        const elementsInLane = path.elements.filter(e =>
          lane.elements.includes(e.id)
        )

        if (elementsInLane.length > 0) {
          const key = `${lane.id}_${path.outcome}`

          if (!grouped[key]) {
            grouped[key] = []
          }

          grouped[key].push(...elementsInLane)
        }
      }
    }

    return grouped
  }
}
```

### Шаг 2: Назначение Y координат путям

```javascript
function assignYCoordinatesToPaths(lane, paths) {
  const centerY = lane.y + lane.height / 2
  const pathCount = paths.length

  if (pathCount === 1) {
    // Один путь - по центру
    return [{ path: paths[0], y: centerY }]
  }

  // Несколько путей - распределить
  const assignments = []
  const offset = PATH_SEPARATION.PREFERRED_OFFSET

  // Главный путь (обычно success) по центру
  const mainPath = paths.find(p => p.isMainPath) || paths[0]
  assignments.push({ path: mainPath, y: centerY })

  // Остальные пути выше и ниже
  const otherPaths = paths.filter(p => p !== mainPath)

  otherPaths.forEach((path, index) => {
    const direction = index % 2 === 0 ? 1 : -1  // Чередовать верх/низ
    const multiplier = Math.floor(index / 2) + 1

    assignments.push({
      path: path,
      y: centerY + (direction * offset * multiplier)
    })
  })

  return assignments
}
```

### Шаг 3: Проверка коллизий

```javascript
function detectCollisions(elements, margin = 20) {
  const collisions = []

  for (let i = 0; i < elements.length; i++) {
    for (let j = i + 1; j < elements.length; j++) {
      const elem1 = elements[i]
      const elem2 = elements[j]

      if (doElementsOverlap(elem1, elem2, margin)) {
        collisions.push({
          element1: elem1.id,
          element2: elem2.id,
          severity: calculateOverlapSeverity(elem1, elem2)
        })
      }
    }
  }

  return collisions
}

function doElementsOverlap(elem1, elem2, margin) {
  const bounds1 = {
    left: elem1.x - margin,
    right: elem1.x + elem1.width + margin,
    top: elem1.y - margin,
    bottom: elem1.y + elem1.height + margin
  }

  const bounds2 = {
    left: elem2.x - margin,
    right: elem2.x + elem2.width + margin,
    top: elem2.y - margin,
    bottom: elem2.y + elem2.height + margin
  }

  return !(bounds1.right < bounds2.left ||
           bounds1.left > bounds2.right ||
           bounds1.bottom < bounds2.top ||
           bounds1.top > bounds2.bottom)
}
```

### Шаг 4: Автокоррекция позиций

```javascript
function autoCorrectPositions(elements, collisions) {
  const corrections = []

  for (const collision of collisions) {
    const elem1 = findElement(elements, collision.element1)
    const elem2 = findElement(elements, collision.element2)

    // Определить как разрешить конфликт
    if (elem1.x === elem2.x) {
      // Одинаковая X - сдвинуть по Y
      const offset = elem1.height + PATH_SEPARATION.PREFERRED_OFFSET

      corrections.push({
        element: elem2.id,
        newY: elem1.y + offset,
        reason: `Collision with ${elem1.id} - moved down`
      })
    } else if (elem1.y === elem2.y) {
      // Одинаковая Y - сдвинуть по X
      const offset = elem1.width + SPACING.HORIZONTAL_STEP

      corrections.push({
        element: elem2.id,
        newX: elem1.x + offset,
        reason: `Collision with ${elem1.id} - moved right`
      })
    }
  }

  return corrections
}
```

## Checklist перед генерацией

```markdown
- [ ] Все пути к разным outcomes имеют разные Y координаты
- [ ] Минимальное расстояние между путями >= 80px
- [ ] Flows используют коридоры при пересечении lanes
- [ ] Flows имеют clearance >= 20px от всех элементов
- [ ] Субпроцессы > 5 элементов свернуты (isExpanded="false")
- [ ] Нет элементов с одинаковыми (x, y) координатами
- [ ] Все flows используют ортогональные линии (90°)
- [ ] Waypoints не проходят через элементы
- [ ] Lane heights учитывают все пути (включая альтернативные)
```

## Примеры "до" и "после"

### До: Наложение End Events

```javascript
// ❌ ПЛОХО
Task_Success:  { x: 2270, y: 110 }
Task_Reject:   { x: 2270, y: 110 }  // Конфликт!
End_Success:   { x: 2442, y: 132 }
End_Reject:    { x: 2442, y: 132 }  // Конфликт!
```

### После: Разделение путей

```javascript
// ✅ ХОРОШО
Task_Success:  { x: 2270, y: 110 }  // Главный путь
Task_Reject:   { x: 2270, y: 210 }  // +100px вниз
End_Success:   { x: 2442, y: 132 }
End_Reject:    { x: 2442, y: 232 }  // +100px вниз
```

### До: Flow пересекает элементы

```xml
<!-- ❌ ПЛОХО -->
<bpmndi:BPMNEdge id="Flow_ChecksFailed">
  <di:waypoint x="965" y="470"/>
  <di:waypoint x="2270" y="210"/>  <!-- Диагональ через все! -->
</bpmndi:BPMNEdge>
```

### После: Flow через коридор

```xml
<!-- ✅ ХОРОШО -->
<bpmndi:BPMNEdge id="Flow_ChecksFailed">
  <di:waypoint x="965" y="470"/>   <!-- Начало -->
  <di:waypoint x="965" y="260"/>   <!-- Вверх к коридору -->
  <di:waypoint x="2220" y="260"/>  <!-- По коридору -->
  <di:waypoint x="2220" y="210"/>  <!-- Вниз к задаче -->
  <di:waypoint x="2270" y="210"/>  <!-- К задаче -->
</bpmndi:BPMNEdge>
```
