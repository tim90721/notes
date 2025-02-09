- [Introduction](#introduction)
- [Spin Lock Usage](#spin-lock-usage)
- [Source Code](#source-code)
  - [Data Structure](#data-structure)
  - [Initialization](#initialization)
# Introduction
# Spin Lock Usage
To use spinlock in kernel. One must first initialize it by either
``` C
DEFINE_SPINLOCK(lock);
```

or
``` C
spin_lock_init(lock);
```

There are several APIs to perform lock and unlock on the spinlock. Each of them
needs to be used on a certain scenario. If use the wrong API on a certain scenario,
it may affect performance or cause deadlock.
> [!NOTE] Explain which API should be used at what scenario

To enter critical section and protect shared data
``` C
spin_lock(lock);
spin_lock_bh(lock);
spin_lock_irq(lock);
spin_lock_irqsave(lock, flag);
```

To leave critical section
``` C
spin_unlock(lock);
spin_unlock_bh(lock);
spin_unlock_irq(lock);
spin_unlock_irqrestore(lock, flag);
```

For example,
``` C
DEFINE_SPINLOCK(lock);

spin_lock(lock);
// enter critical section
// operate shared data
// leave critical section
spin_unlock(lock);
```

# Source Code
## Data Structure
Generic spinlock_t structure is defined as the following
``` C
/* Non PREEMPT_RT kernels map spinlock to raw_spinlock */
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;
/* include/linux/spinlock_types.h */
```
Ignore the debug part, it is actually just a raw_spinlock structure.
> [!NOTE] Why raw_spinlock?
> In the early version, it actually only has 1 type of spinlock which 
> is the legacy busy-wait lock. 
> 
> Since the preemptive realtime kernel development, spinlock needs further
> enhancement to fulfill realtime requirements. In Linux realtime development branch
> (which is actaully merged into mainline recently), the spinlock is actually a
> mutex which supports priority inversion. We may talk about that in the future.
> However, right now, I just want to talk about basic spinlock.

raw_spinlock structure definition.
``` C
typedef struct raw_spinlock {
	arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
/* include/linux/spinlock_types_raw.h */
```

arch_spinlock_t is defined in architecture dependent spinlock header file. Here we 
take arm64 architecture for example.
``` C
#include <asm-generic/qspinlock_types.h>
/* arch/arm64/include/asm/spinlock_types.h */
```
In arm architecture, spinlock is a simple busy-wait lock at the beginning. However,
as the time goes by, to improve the performance and fairness, ticket based spinlock
replaced raw busy-wait lock for a while. Later, ticket based spinlock was replaced by
qspinlock. Things just get more complicated but have more performance and fairness
improvement. 

We will start from the generic API first and easiest busy-wait lock first. Then we
can dive to ticket based lock. Finally, We will talk about qspinlock since this is
the current qspinlock implementation that arm64 applied.

## Initialization
The static way is `DEFINE_SPINLOCK`
``` C
#define ___SPIN_LOCK_INITIALIZER(lockname)	\
	{					\
	.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,	\
	SPIN_DEBUG_INIT(lockname)		\
	SPIN_DEP_MAP_INIT(lockname) }

#define __SPIN_LOCK_INITIALIZER(lockname) \
	{ { .rlock = ___SPIN_LOCK_INITIALIZER(lockname) } }

#define __SPIN_LOCK_UNLOCKED(lockname) \
	(spinlock_t) __SPIN_LOCK_INITIALIZER(lockname)

#define DEFINE_SPINLOCK(x)	spinlock_t x = __SPIN_LOCK_UNLOCKED(x)
/* include/linux/spinlock_types.h */
```

The way to do spinlock dynamic initialization is `spin_lock_init`
``` C
#define __RAW_SPIN_LOCK_INITIALIZER(lockname)	\
{						\
	.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,	\
	SPIN_DEBUG_INIT(lockname)		\
	RAW_SPIN_DEP_MAP_INIT(lockname) }

#define __RAW_SPIN_LOCK_UNLOCKED(lockname)	\
	(raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)

#define DEFINE_RAW_SPINLOCK(x)  raw_spinlock_t x = __RAW_SPIN_LOCK_UNLOCKED(x)
/* include/linux/spinlock_types_raw.h */

# define raw_spin_lock_init(lock)				\
	do { *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock); } while (0)

static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}

#ifdef CONFIG_DEBUG_SPINLOCK

# define spin_lock_init(lock)					\
do {								\
	static struct lock_class_key __key;			\
								\
	__raw_spin_lock_init(spinlock_check(lock),		\
			     #lock, &__key, LD_WAIT_CONFIG);	\
} while (0)
/* include/linux/spinlock.h */
```