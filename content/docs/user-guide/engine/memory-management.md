---
description: ' Learn best practices for memory management in Open 3D Engine. '
title: Memory Management
---

When managing memory in O3DE, use AZ memory management calls and avoid static variables whose constructors allocate memory or connect to EBuses.

## Memory Allocations

When allocating memory, use the following recommended practices:
+ Do not use `new`, `malloc`, and similar allocators directly. Instead, use AZ memory manager calls to `aznew`, `azmalloc`, `azfree`, `azcreate`, and `azdestroy`.
+ Specify the allocator in each class.
+ Use child allocators. To tag and track resource usage, new gems and subsystems should create their own allocator or create a `ChildAllocator` that references an existing allocator. For an example of creating a child allocator, see [Creating an Allocator](/docs/user-guide/engine/memory/allocators/).

**Reason**: O3DE's core AZ systems provide a memory managed environment, not a raw system allocator like the managed memory in C\# or Java. In O3DE, the core AZ systems provide speed, safety, and facilities for tracking memory usage.

For information about O3DE's scheme for allocators and new allocators, see [Manually Allocating Memory](/docs/user-guide/engine/memory/allocators/).

## Memory Issues Caused by Static Variables

In constructors and destructors for static variables, avoid the following:
+ Allocating memory with the AZ memory allocator system.
+ Deallocating memory with the AZ memory allocator system.
+ Connecting to EBuses
+ Disconnecting from EBuses.

These rules apply to global static variables, function local static variables, and containers.

**Reason:** O3DE manages the memory of its classes and container types using the AZ memory allocator system. However, class static variables and containers can attempt to use the AZ memory allocator system before it is created or after it no longer exists. Keep in mind the following:
+ On application startup or when a gem is loaded, if a static variable attempts to use the AZ system allocator before the AZ system allocator has been created, a crash can occur.
+ On shutdown, if a static variable attempts to deallocate from the AZ system heap when the AZ system allocator no longer exists, a crash can occur.

Remember that the lifetime of a static variable lasts until the module in which the variable is declared is unloaded. In static linked libraries, the duration of static variables is the entire duration of the application.

### Static Local Variable Example

The following example code from AzFramework \(`\dev\Code\Framework\AzFramework\AzFramework\Physics\DefaultDebugDrawSettings.h`\) uses function static container variables whose destructors rely on `AZ::SystemAllocator`.

**Example**

```
namespace Physics
{
    AZ_INLINE void GetDefaultDebugDrawSettings(DebugDrawSettings& settings)
    {
        static AZStd::vector<Vec3> verts;      // These function static container variables use AZ::SystemAllocator.
        static AZStd::vector<ColorB> colors;
        //...
    }
}
```

By default, the `AZStd::vector` class uses the `SystemAllocator`. Invoking this function on application shutdown, when `SystemAllocator` no longer exists, causes a deletion from a destroyed heap.

### Global Static Variable Example 

On startup, the following code attempts to create a global static variable whose constructor connects to an EBus:

**Example**

```
// Example of a global static variable that connects to an EBus.
namespace LmbrCentral
{
    // Wraps an IMaterial pointer so that BehaviorContext can use it.
    class MaterialHandle
        : public AZ::RenderNotificationsBus::Handler
    {
    public:
        AZ_CLASS_ALLOCATOR(MaterialHandle, AZ::SystemAllocator, 0);
        AZ_TYPE_INFO(MaterialHandle, "{BF659DC6-ACDD-4062-A52E-4EC053286F4F}");
        MaterialHandle()
        {
            AZ::RenderNotificationsBus::Handler::BusConnect();
        }
//...
// Later in the code, a MaterialHandle in global space is declared:
MaterialHandle g_defaultMaterialHandle;
```

The code crashes the engine as soon as the module that contains the global variable definition is loaded. The constructor of the global variable attempts to connect to the EBus, but the EBus system is not ready because it uses an environment variable from a module that has not yet been initialized.
