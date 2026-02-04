# Передвижение по планете (сферический мир)

Ниже — практическая схема, которую можно адаптировать под Unity/Unreal/Godot или собственный движок.

## 1) Базовая геометрия

- Планета — сфера с радиусом `R` и центром `C`.
- Позиция игрока: `P`.
- Вектор «вверх» для игрока: `Up = normalize(P - C)`.
- Вектор «вниз» (гравитация): `Down = -Up`.

Это автоматически даёт локальную вертикаль в любой точке поверхности.

## 2) Гравитация

В физическом апдейте:

1. Посчитать `Down`.
2. Применить ускорение: `velocity += Down * gravity * dt`.
3. Корректировать ориентацию игрока, чтобы локальная `up` совпадала с `Up`.

### Вариант с «липкой» гравитацией

Чтобы избежать отрыва от поверхности при небольших скачках:

- если игрок слишком далеко от поверхности, «притягивать» к ней с мягкой пружиной
- либо использовать **Character Controller** и «прижимать» вниз небольшим постоянным ускорением

## 3) Движение в касательной плоскости

Ключевое правило: горизонтальная скорость должна быть **перпендикулярна `Up`**.

Алгоритм:

1. Рассчитать `Up`.
2. Взять входной вектор `Input` в локальных координатах игрока (например, WASD в плоскости камеры).
3. Спроецировать его на касательную плоскость:

```
Tangent = Input - Up * dot(Input, Up)
Tangent = normalize(Tangent)
```

4. Применить скорость: `velocity += Tangent * moveSpeed`.

## 4) Ориентация персонажа

Для «натурального» поведения:

- `Up` совпадает с локальной осью `Y` персонажа.
- «Вперёд» (`Forward`) лежит в касательной плоскости.

Способ:

```
Forward = normalize(Forward - Up * dot(Forward, Up))
Right = normalize(cross(Up, Forward))
Forward = cross(Right, Up)
```

## 5) Камера

Чтобы камера не «заваливалась»:

- целевая «вертикаль» камеры также выравнивается по `Up`.
- позиция камеры рассчитывается по смещению относительно персонажа, но в локальной системе, заданной `Up`.

Пример:

```
camPos = playerPos - Forward * dist + Up * height
camLook = playerPos + Up * lookOffset
```

## 6) Коллизии и поверхность

Для качественного движения:

- используйте физический «капсульный» коллайдер;
- проверяйте поверхность Raycast-ом по `Down` и используйте нормаль поверхности;
- если планета не идеально сферическая (например, процедурная), `Up` нужно брать из нормали поверхности.

## 7) Псевдокод (движение за тик)

```
Up = normalize(P - C)
Down = -Up

// gravity
velocity += Down * gravity * dt

// input
Input = getMoveInputRelativeToCamera()
Tangent = Input - Up * dot(Input, Up)
if (length(Tangent) > 0):
  Tangent = normalize(Tangent)
  velocity += Tangent * moveSpeed

// position update
P += velocity * dt

// orientation
alignPlayerUpTo(Up)
```

## 8) Чем Spore и DSP отличаются от «обычной сферы»

1. **Планета иногда не идеальна** (детализированный рельеф), поэтому лучше опираться на **нормаль поверхности**.
2. **Камера держит горизонт** за счёт постоянного выравнивания по локальному `Up`.
3. **Переход в космос**: при удалении от поверхности постепенно снижать «липкость» и переключать управление на свободное.

---

## 9) Пример под Unity (CharacterController)

Ниже — минимальный пример скрипта, который:

- удерживает игрока на сфере;
- выравнивает локальный `up` по нормали к центру планеты;
- проецирует ввод в касательную плоскость;
- корректно ведёт камеру от третьего лица.

### Скрипт: `PlanetWalker.cs`

```csharp
using UnityEngine;

[RequireComponent(typeof(CharacterController))]
public class PlanetWalker : MonoBehaviour
{
    [Header("Planet")]
    public Transform planetCenter;
    public float gravity = 9.81f;

    [Header("Movement")]
    public float moveSpeed = 6f;
    public float jumpSpeed = 6f;
    public float groundStickForce = 5f;

    [Header("Camera")]
    public Transform cameraPivot;
    public float cameraDistance = 6f;
    public float cameraHeight = 2f;
    public float cameraLookOffset = 1.2f;

    private CharacterController controller;
    private Vector3 velocity;

    private void Awake()
    {
        controller = GetComponent<CharacterController>();
    }

    private void Update()
    {
        if (planetCenter == null)
        {
            Debug.LogWarning("PlanetWalker: planetCenter is not set.");
            return;
        }

        Vector3 up = (transform.position - planetCenter.position).normalized;
        Vector3 down = -up;

        // Align player up with planet normal.
        Quaternion targetRotation = Quaternion.FromToRotation(transform.up, up) * transform.rotation;
        transform.rotation = Quaternion.Slerp(transform.rotation, targetRotation, 10f * Time.deltaTime);

        // Movement input relative to camera.
        Vector2 input = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));
        Vector3 camForward = Vector3.ProjectOnPlane(cameraPivot.forward, up).normalized;
        Vector3 camRight = Vector3.ProjectOnPlane(cameraPivot.right, up).normalized;
        Vector3 desired = (camForward * input.y + camRight * input.x);

        if (desired.sqrMagnitude > 1f)
        {
            desired.Normalize();
        }

        // Tangent movement.
        Vector3 tangent = Vector3.ProjectOnPlane(desired, up);
        Vector3 move = tangent * moveSpeed;

        // Gravity + jump.
        if (controller.isGrounded)
        {
            velocity = Vector3.ProjectOnPlane(velocity, up);

            if (Input.GetButtonDown("Jump"))
            {
                velocity += up * jumpSpeed;
            }
            else
            {
                // Small downward force to keep grounded.
                velocity += down * groundStickForce;
            }
        }

        velocity += down * gravity * Time.deltaTime;

        // Apply movement.
        Vector3 finalMove = (move + velocity) * Time.deltaTime;
        controller.Move(finalMove);

        UpdateCamera(up);
    }

    private void UpdateCamera(Vector3 up)
    {
        if (cameraPivot == null)
        {
            return;
        }

        Vector3 forward = Vector3.ProjectOnPlane(transform.forward, up).normalized;
        Vector3 targetPos = transform.position - forward * cameraDistance + up * cameraHeight;
        cameraPivot.position = Vector3.Lerp(cameraPivot.position, targetPos, 10f * Time.deltaTime);
        cameraPivot.rotation = Quaternion.LookRotation((transform.position + up * cameraLookOffset) - cameraPivot.position, up);
    }
}
```

### Настройка в Unity

1. **Создать планету**: объект-сфера (или меш), поставить в `planetCenter`.
2. **Игрок**: объект с `CharacterController` и `PlanetWalker`.
3. **Камера**: создать `cameraPivot` (пустой объект), внутрь него — `Camera`, задать ссылку в скрипте.
4. **Input**: оси `Horizontal`/`Vertical` и кнопка `Jump` должны быть настроены (по умолчанию в Input Manager они есть).

### Примечания

- Для неровного рельефа используйте нормаль поверхности (Raycast по `down`) вместо `up = (P - C).normalized`.
- Для больших скоростей и прыжков может понадобиться увеличить `CharacterController.stepOffset` и `skinWidth`.
- Если камера «дёргается» — сгладьте `cameraPivot.position` и/или используйте `LateUpdate` для камеры.

---

Если нужны примеры под другие движки (Unreal/Godot) или ECS-реализация под Unity DOTS, можно добавить отдельные файлы.
