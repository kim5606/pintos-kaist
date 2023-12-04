Brand new pintos for Operating Systems and Lab (CS330), KAIST, by Youngjin Kwon.

The manual is available at https://casys-kaist.github.io/pintos-kaist/.

------------
# WIL WEEK 7

## Team schedule

### 팀 진행 스케쥴
- 24 ~ 26 alarm clock
- 26 ~ 30 priority problem
  - 26 ~ 27 priority scheduling
  - 28 ~ 29 scheduling sync , inversion problem

### 진행 방식
1. Daily 
- 오전 10:00
  - 코드 구현 시 어려웠던 점 5분 발표, PULL
- 오후 9:00
  - 코드 리뷰, PUSH
2. Rule
- pintos 저자가 작성한 듯이, 코딩 컨벤션 지키면서 작성
  - 관련 있는 곳에 함수 만들기
  - 전역 변수 사용 주의
  - 실행 흐름 파악하고 유지하기
  - 네이밍 신경쓰기
- 실행 잘 되고, 가독성 좋은 코드로 통일
  - 각자 진행하면서 힘들었던 부분 공유

## Alarm clock
기존 pintos에서 알람 기능은 'busy-waiting' 상태이다.
이를 방지하고자 sleep queue를 만들고 거기에 스레드를 재운 뒤(block), 일정 시간이 되면 스레드를 깨우는 방식을 진행한다.

### 수정한 함수

timer interrupt
```c
// thread.h 파일: thread 구조체에 wake up tick 필드 추가
/* wake up tick 추가 */
struct thread {
    ...
    int64_t wake_up_tick;   // Thread가 깨어나야 하는 tick
    ...
};
```

```c
// thread.c 파일: 전역 변수 추가 및 초기화
/* 전역 변수 추가 */
static struct list sleep_list;       // Sleep 중인 스레드 목록
static int64_t next_tick_to_awake;   // 다음에 깨어날 스레드의 tick

/* thread_init() 함수 수정 */
void
thread_init (void) {
    ...
    list_init (&sleep_list);         // Sleep list 초기화
    ...
}
```

```c
// thread.h 파일: 새로운 함수 선언
/* 구현 함수 선언 */
void thread_sleep(int64_t ticks);
void thread_awake(int64_t ticks);
void update_next_tick_to_awake(int64_t ticks);
int64_t get_next_tick_to_awake(void);
```

```c
// timer.c 파일: timer_sleep() 함수 구현
/* timer_sleep() 함수 구현 */
void
timer_sleep (int64_t ticks) {
    int64_t start = timer_ticks ();
    ASSERT (intr_get_level () == INTR_ON);
    thread_sleep (start + ticks);
}
```

```c
// thread.c 파일: thread_sleep() 함수 구현
/* thread_sleep() 함수 구현 */
void
thread_sleep (int64_t ticks) {
    struct thread *current_thread = thread_current ();
    enum intr_level old_level;

    old_level = intr_disable (); // 인터럽트 비활성화

    ASSERT (current_thread != idle_thread);

    current_thread->wake_up_tick = ticks;
    list_push_back (&sleep_list,&current_thread->elem); // Sleep list에 추가
    thread_block (); // 스레드를 block 상태로 변경

    intr_set_level (old_level); // 인터럽트 원래 상태로 복원
}
```

```c
// timer.c 파일: timer_interrupt 함수 수정
/* timer_interrupt 수정 */
static void
timer_interrupt (struct intr_frame *args UNUSED) {
    ticks++;
    thread_tick ();
    if (ticks >= next_tick_to_awake)
        thread_wakeup (ticks);
}
```

```c
// thread.c 파일: thread_wakeup 함수 구현
/* thread_wakeup 수정 */
void
thread_wakeup (int64_t current_ticks) {
    enum intr_level old_level;
    old_level = intr_disable ();

    while (!list_empty (&sleep_list)) {
        struct thread *t = list_entry (list_front (&sleep_list), struct thread, elem);
        if (t->wake_up_tick <= current_ticks) {
            list_pop_front (&sleep_list);  // Sleep list에서 제거
            thread_unblock (t);           // 스레드를 unblock 상태로 변경
        } else {
            break;
        }
    }
    intr_set_level (old_level); // 인터럽트 원래 상태로 복원
}
```

## Priority Scheduling
스레드는 이제 실행에 대한 우선권(=Priority)를 가지고 스케쥴러는 Priority에 따라 실행할 스레드를 결정하게 된다. 이를 구현하기 위해서는 크게 두가지 : sorting&preemption이 필요하다. 그리고 Priority Scheduling에 따른 Priority Inversion의 문제를 해결하기 위해 Priority Donation이 필요하다.
### Sorting for thread queue
현재 실행중이지 않은 스레드는 다양한 큐(ex. ready queue, sema waiters ...)에 대기중일 수 있다. 이 때 우선순위 순서대로 스레드를 큐에 삽입하여, 우선순위가 가장 높은 스레드가 대기중인 큐에서 먼저 빠져나올 수 있도록 보장해주는 것이다. 리스트를 전체 순회하여 우선순위가 가장 높은 스레드를 찾아도 되지만, 코드의 가독성과 효율성을 위해서 굳이 그렇게할 필요는 없어보인다.
### Preemption
새로운 스레드의 생성, 기존 스레드의 Priority 변경, Sleep 되었던 스레드가 깨어나는 등 다양한 케이스에 의해 Ready Queue에서 대기중인 스레드의 우선순위가 현재 실행중인 스레드의 우선순위보다 높을 가능성이 있다. 이 경우에 스레드가 CPU를 '선점'하는 로직을 추가 하여, 가장 높은 Priority를 가진 스레드가 CPU를 할당 받고 있도록 보장한다.

## Priority Inheritance
우선 순위 스케줄링을 수행할 때 주의할 점이 있습니다. 동기화를 수행할 때 우선 순위가 높은 프로세스 $H$가 우선 순위가 낮은 프로세스 $L$를 대기하고 있는 상황에서 $H$과 $L$의 사이의 우선 순위를 가진 프로세스 $M_1$, $M_2$, $M_3, ...$가 계속해서 들어오면, 우선 순위가 높은 프로세스 $H$는 작업을 수행하지 못하는 문제가 있습니다.

이를 해결하기 위해서 몇가지 방법을 생각할 수 있습니다.
1. 비선점 스케줄링
2. 우선 순위 상속 적용
3. 주기적으로 우선 순위를 최대로 높임
4. 허용된 소비 시간을 기준으로 우선 순위를 결정

1번 방법은 문제를 해결한다기보다, 문제 상황을 제거하는 방법이기 때문에 고려하지 않겠습니다. 따라서 일반적으로는 2번, 3번, 4번 중 하나를 선택하게 됩니다. 3번과 4번의 경우, 실시간 시스템에서 우선 순위가 높은 일을 빠르게 처리해야 하기 때문에 해당 방법들을 곧장 적용하기는 어렵습니다.

가장 빠르고 직관적인 해결 방법은 우선 순위 상속입니다. 우선 순위 상속이란 상기 예제에서 프로세스 $H$의 우선 순위를 lock을 가진 프로세스 $L$에게 넘기는 것입니다. 그리고 임계 영역에서 빠져나오면 $L$은 원래의 우선 순위로 복귀합니다.

### 상황 1: 하나의 프로세스가 여러 lock을 홀딩
이 경우는 자신과 프로세스가 홀딩하고 있는 lock을 기다리고 있는 다른 프로세스 중에서 가장 큰 값을 선택하면 됩니다.

### 상황 2: 재귀적 lock
이 경우는 재귀적으로 lock이 걸려있는 상황입니다. 프로세스 $A$가 $l_1$을 가지고 있고, 프로세스 $B$가 $l_2$를 가지고 $l_1$을 대기하고 있는 상황에서 프로세스 $C$가 $l_2$를 대기하고 있는 경우입니다.

이 경우에는 재귀적으로 타고 내려가면서 현재 재귀에서 가장 큰 우선 순위를 전파합니다.

상황 1과 상황 2를 동시에 고려하면 우선 순위 상속을 구현할 수 있습니다.

#### lock_acquire
``` c
void
lock_acquire (struct lock *lock) {
	enum intr_level old_level;
	struct thread *curr;
	struct thread *holder;
	int max_priority;
	
	ASSERT (lock != NULL);
	ASSERT (!intr_context ());
	ASSERT (!lock_held_by_current_thread (lock));
	
	old_level = intr_disable ();
	curr = thread_current ();
	holder = lock->holder;
	max_priority = PRI_MIN;
	
	if (holder) {
		curr->lock = lock;
		list_insert_ordered (&holder->donations, &curr->delem, 
						cmp_thrd_donation_priorities, NULL);
		max_priority = curr->priority;

		struct thread *cursor = holder;
		while (cursor->lock != NULL) {
			if (max_priority < cursor->priority) {
				max_priority = cursor->priority;
			}

			cursor->priority = max_priority;
			cursor = cursor->lock->holder;
		}

		if (cursor != NULL) {
			cursor->priority = max_priority;
		}

		holder->priority = max_priority;
	}

	sema_down (&lock->semaphore);
	intr_set_level (old_level);
	curr->lock = NULL;
	lock->holder = thread_current ();
}

```

#### lock_release
``` c
void
lock_release (struct lock *lock) {
	struct thread *holder;
	int max_priority;

	ASSERT (lock != NULL);
	ASSERT (lock_held_by_current_thread (lock));

	holder = lock->holder;
	holder->priority = holder->prev_priority;
	for (struct list_elem *e = list_begin (&holder->donations);
		e != list_end (&holder->donations);) {

		struct thread* t = list_entry (e, struct thread, delem);
		if (t->lock == lock) {
			e = list_remove (e);
			continue;
		}

		e = list_next (e);
	}

	if (!list_empty (&holder->donations)) {
		list_sort (&holder->donations, cmp_thrd_donation_priorities, NULL);
		max_priority = list_entry (list_begin (&holder->donations), 
					struct thread, delem)->priority;
		if (holder->priority < max_priority) {
			holder->priority = max_priority;
		}
	}

	lock->holder = NULL;
	sema_up (&lock->semaphore);
}
```

## Try-Errors of week
### 김의훈
1. 코딩 컨벤션
- pintos 저자의 코딩 컨벤션 형식에 맞춰야 했기 때문에, 코드를 복기할 시간이 있고 주석 또한 그들의 시점에서 생각하면서 달 수 있음

2. 몇몇 함수 이해 문제
- 처음에 아예 이해가 되지 않아 해당 구조를 파악하는데 팀원 도움을 받고 그림을 최대한 많이 그려보고 이해하려고 노력
```c
bool
cmp_sem_priority (const struct list_elem *a, const struct list_elem *b, void *aux) {

	struct semaphore_elem * sa = list_entry(a, struct semaphore_elem, elem); 
	struct semaphore_elem * sb = list_entry(b, struct semaphore_elem, elem);
	

	struct list_elem *sa_e = list_begin(&(sa->semaphore.waiters));
	struct list_elem *sb_e = list_begin(&(sb->semaphore.waiters));

	struct thread *sa_t = list_entry(sa_e, struct thread, elem);
	struct thread *sb_t = list_entry(sb_e, struct thread, elem);
	return (sa_t->priority) > (sb_t->priority);
}
```

```c
void
lock_acquire (struct lock *lock) {
	ASSERT (lock != NULL);
	ASSERT (!intr_context ());
	ASSERT (!lock_held_by_current_thread (lock));
	
	struct thread *cur = thread_current();

	if (lock->holder) { 				
		cur -> wait_on_lock = lock;									
		list_insert_ordered(&lock->holder->donations,&cur->donation_elem,
		thread_compare_donate_priority,0);							
		donate_priority ();										
	}

	sema_down (&lock->semaphore);
	cur->wait_on_lock = NULL;
	lock->holder = thread_current ();	
}
```

3. 디버깅 문제
- 커널 패닉이 일어나면 디버깅을 하는 과정에서 오류 포인트를 정확하게 검출하는게 어려움이 있음

### 이병재
1. 주석은 영어로 작성 : 
핀토스 전체의 코딩 컨벤션과 통일하기 위해
```c
// Private note : call sort function, ensuring that donor_list is sorted by priority
list_sort (&curr->donor_list, &cmp_priority_greater_dona, NULL);
```
2. Null 포인터에 대한 잘못된 참조 : 
Donor->Lock_Waiting->Holder에서 Lock_Waiting이 Null인 경우를 고려하지 않아서 커널 패닉 발생. 두번이상 타고 들어갈 때는 Null체크를 꼭 하는 것으로 변경
```c
for (int i = 0; i < max_nested_depth; i++) {
  donee->priority = donor->priority;
  
  donor = donee;
  if (!donor->lock_waiting) 
    break;
  donee = donor->lock_waiting->holder;
}
```
3. 효율성 vs 안정성 : 
Queue에서 Thread를 Insert/Extract할 때 효율을 위해서 정렬을 한번만 하면 된다고 생각함. 그런데 중간에 예상치 못한 케이스가 있었음(ex. nested donation에 따른 priority 변경). 이후 안전성을 위해 중간에 어떤 과정이 생기더라도 insert/extract 모두 sort를 하는 방향으로 코드를 짜는 것이 좋겠다고 생각
4. 인터럽트를 끄고 키는 것에 대한 이해 부족 : 
스레드의 안전한 작업을 보장해야 하는 코드 블록의 경우 인터럽트를 끈 후 코드를 실행하고 이후에 원래 레벨로 복원해야 하는데, 아직 확실하게 이해하고 있는 것은 아님. 프로젝트를 진행하면서 계속 익혀볼 예정.

### 조정민
#### 1. List Remove
리스트에서 원소를 제거하는 `list_remove`는 아래와 같이 구현되었습니다.
``` c
struct list_elem *
list_remove (struct list_elem *elem) {
	ASSERT (is_interior (elem));
	elem->prev->next = elem->next;
	elem->next->prev = elem->prev;
	return elem->next;
}
```
`elem`를 삭제하고 `elem->next`를 반환함에 주의해야합니다. 따라서 아래와 같이 리스트에서 원소를 삭제하는 전통적인 루프는 작동하지 않습니다.

``` c
struct list_elem *e = list_begin (&list);
for (; e != list_end (&list); e = list_next (e)) {
	e = list_remove (e);
}
```
따라서 아래와 같이 해야합니다.
``` c
struct list_elem *e = list_begin (&list);
for (; e != list_end (&list);) {
	if (predicate) {
		e = list_remove (e);
		continue;
	}
	e = list_next (e);
}
```

#### 2. Interrupt
언제 인터럽트를 걸어야하는 지 판단하는데 어려움을 겪었습니다. 이를 해결하기 위해서 문서를 조금 참조했습니다.

##### 인터럽트의 종류
인터럽트는 내부 인터럽트와 외부 인터럽트로 나뉩니다. 내부 인터럽트는 CPU 인터럽트에 의해서 발생하는 인터럽트이며 동기식입니다. 예를 들어 시스템 콜, 페이지 오류, 0으로 나누는 행위들은 내부 인터럽트를 발생시킵니다. `intr_disable()`을 사용해도 내부 인터럽트는 꺼지지 않습니다. 외부 인터럽트는 CPU 외부에서 발생하는 인터럽트입니다. 타이머 인터럽트, 키보드 입력, 디스크 읽기 등 하드웨어 작업에서 발생합니다. 외부 인터럽트는 비동기식이고, 인터럽트를 잠시 끌 수 있습니다.

##### 인터럽트 끄기
문제 상황은 현재 실행 흐름을 멈추고 인터럽트 핸들러에서 인터럽트를 처리한 뒤 다른 스레드로 문맥 전환이 일어나는 경우 발생합니다. 이 때 두 쓰레드가 같은 공유 자원에 접근할 경우 문제가 생길 수 있습니다. 이러한 경우는 안전합니다.

``` c
void safe () {
	struct thread *curr = thread_current ();

	// Working with curr.
	// It is okay to call current_thread () multiple times within this context.
}
```

일반적으로 이러한 컨텍스트는 보존되기 때문에 괜찮습니다. 하지만 전역변수나 다른 쓰레드의 정보를 수정하는 경우에는 인터럽트를 꺼야합니다. 예를 들어 `ready_list`에 원소를 집어넣거나 빼는 경우는 안전하지 않을 가능성이 있습니다. 기존 코드를 분석에서 `ready_list`를 수정하는 모든 작업들에서 인터럽트를 끄고 있음을 관찰할 수 있습니다. 이를 작성일(2023.11.29)에 발견하였고, 전역 변수에 접근하는 모든 코드에 대해서 인터럽트를 끌 예정입니다.
