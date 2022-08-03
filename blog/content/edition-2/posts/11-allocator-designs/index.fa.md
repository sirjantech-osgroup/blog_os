+++
title = "Allocator Designs"
weight = 11
path = "allocator-designs"
date = 2020-01-20

[extra]
chapter = "Memory Management"
# Please update this when updating the translation
translation_based_on_commit = "aa389dae4b0b756a13ae3c330fa9bdf55d5f5ba"
# GitHub usernames of the people that translated this post
translators = ["hamidrezakp", "MHBahrampour"]
rtl = true
+++

این پست نحوه پیاده‌سازی تخصیص‌دهنده هیپ را از ابتدا توضیح می‌دهد. این طرح‌های مختلف تخصیص‌دهنده، از جمله تخصیص بامپ (bump)، تخصیص لیست پیوندی (linked list)، و تخصیص بلوک با اندازه ثابت (fixed-size block) را ارائه و مورد بحث قرار می‌دهد. برای هر یک از این سه طرح، یک پیاده‌سازی پایه ایجاد می‌کنیم که می‌تواند برای هسته استفاده شود.

<!-- more -->

این بلاگ بصورت آزاد روی [گیت‌هاب] توسعه داده شده است. اگر شما مشکل یا سوالی دارید، لطفاً آن‌جا یک ایشو باز کنید. شما همچنین می‌توانید [در زیر] این پست کامنت بگذارید. منبع کد کامل این پست را می‌توانید در بِرَنچ [`post-11`][post branch] پیدا کنید.

[گیت‌هاب]: https://github.com/phil-opp/blog_os
[در زیر]: #comments

<!-- fix for zola anchor checker (target is in template): <a id="comments"> -->
[post branch]: https://github.com/phil-opp/blog_os/tree/post-11

<!-- toc -->

## مقدمه

در [پست قبلی] پشتیبانی اولیه برای تخصیص هیپ به هسته خود اضافه کردیم. برای آن، ما [یک منطقه حافظه جدید][map-heap] در جداول صفحه ایجاد کردیم و از [از کریت `linked_list_allocator`][use-alloc-crate] برای مدیریت آن حافظه استفاده کردیم. در حالی که ما اکنون یک هیپ داریم که به درستی کار می‌کند، بیشتر کار را به جعبه تخصیص‌دهنده واگذار کردیم بدون اینکه سعی کنیم بفهمیم چگونه کار می‌کند.

[پست قبلی]: @/edition-2/posts/10-heap-allocation/index.md
[map-heap]: @/edition-2/posts/10-heap-allocation/index.md#creating-a-kernel-heap
[use-alloc-crate]: @/edition-2/posts/10-heap-allocation/index.md#using-an-allocator-crate

در این پست، ما نشان خواهیم داد که چگونه به جای تکیه بر یک جعبه تخصیص‌دهنده موجود، تخصیص‌دهنده هیپ خود را از ابتدا ایجاد کنیم. ما در مورد طرح‌های تخصیص‌دهنده مختلف، از جمله یک _تخصیص‌دهنده بامپ_ ساده و یک _تخصیص‌دهنده بلوک با اندازه ثابت_ بحث خواهیم کرد، و از این دانش برای پیاده‌سازی یک تخصیص‌دهنده با عملکرد بهبودیافته (در مقایسه با جعبه `linked_list_allocator`) استفاده خواهیم کرد.

### اهداف طراحی

مسئولیت تخصیص‌دهنده، مدیریت حافظه هیپ موجود است. باید حافظه استفاده نشده را در فراخوانی‌های «alloc» برگرداند و حافظه آزاد شده توسط «dealloc» را پیگیری کند تا بتوان دوباره از آن استفاده کرد. مهمتر از همه، هرگز نباید حافظه‌ای را که در حال حاضر در جای دیگری استفاده می‌شود را در اختیار دیگران قرار دهد زیرا این امر باعث رفتار نامشخص می‌شود.

جدای از صحت آن، اهداف ثانویه زیادی برای این طراحی وجود دارد. به عنوان مثال، تخصیص‌دهنده باید به طور موثر از حافظه موجود استفاده کند و میزان [_تکه‌تکه شدن_] را پایین نگه دارد. علاوه‌بر این، باید برای برنامه‌های همزمان و مقیاس‌پذیری به هر تعداد پردازنده خوب کار کند. برای حداکثر کارایی، حتی می‌تواند چیدمان (layout) حافظه را با توجه به حافظه‌های نهان CPU برای بهبود [cache locality] و جلوگیری از [false sharing] بهینه کند.

[cache locality]: https://www.geeksforgeeks.org/locality-of-reference-and-cache-operation-in-cache-memory/
[_تکه‌تکه شدن_]: https://en.wikipedia.org/wiki/Fragmentation_(computing)
[false sharing]: https://mechanical-sympathy.blogspot.de/2011/07/false-sharing.html

این الزامات می‌تواند تخصیص‌دهنده‌های خوب را بسیار پیچیده کند. برای مثال، [jemalloc] بیش از 30000 خط کد دارد. این پیچیدگی اغلب در کد هسته نامطلوب است، جایی که یک باگ می‌تواند منجر به آسیب‌پذیری‌های امنیتی شدید شود. خوشبختانه، الگوهای تخصیص کد هسته اغلب در مقایسه با کد فضای کاربر بسیار ساده‌تر است، به طوری که طرح‌های تخصیص‌دهنده نسبتا ساده اغلب کافی هستند.

[jemalloc]: http://jemalloc.net/

در ادامه سه طرح احتمالی تخصیص‌دهنده هسته را ارائه می‌کنیم و مزایا و معایب آن‌ها را توضیح می‌دهیم.

## تخصیص‌دهنده بامپ

ساده‌ترین طراحی تخصیص‌دهنده یک _bump allocator_ است (همچنین به عنوان _stack allocator_ شناخته می‌شود). حافظه را به صورت خطی اختصاص می‌دهد و فقط تعداد بایت‌های اختصاص داده شده و تعداد تخصیص‌ها را پیگیری می‌کند. این فقط در موارد استفاده بسیار خاص مفید است زیرا محدودیت شدیدی دارد: فقط می‌تواند تمام حافظه را یک‌جا آزاد کند.

### ایده

ایده پشت تخصیص‌دهنده بامپ تخصیص خطی حافظه با افزایش (یا همان _"bumping"_) متغیر `next` است که به ابتدای حافظه استفاده نشده اشاره می‌کند. در ابتدا، «next» برابر با آدرس شروع هیپ است. در هر تخصیص، `next` با تخصیص افزایش می‌یابد به طوری که همیشه به مرز بین حافظه استفاده شده و استفاده نشده اشاره می‌کند:

![The heap memory area at three points in time:
 1: A single allocation exists at the start of the heap; the `next` pointer points to its end
 2: A second allocation was added right after the first; the `next` pointer points to the end of the second allocation
 3: A third allocation was added right after the second one; the `next pointer points to the end of the third allocation](bump-allocation.svg)
 
اشاره‌گر `next` فقط در یک جهت حرکت می‌کند و بنابراین هرگز یک منطقه حافظه را دو بار به شما تحویل نمی‌دهد. هنگامی که به انتهای هیپ می‌رسد، دیگر حافظه قابل تخصیص نیست و در نتیجه در تخصیص بعدی خطای خارج از حافظه (out-of-memory) رخ می‌دهد.

یک تخصیص‌دهنده بامپ اغلب با یک شمارنده تخصیص پیاده‌سازی می‌شود که در هر فراخوانی «alloc» یک واحد افزایش می‌یابد و در هر فراخوانی «dealloc» یک واحد کاهش می‌یابد. هنگامی که شمارنده تخصیص به صفر می‌رسد به این معنی است که همه تخصیص‌ها در پشته تخصیص داده شده‌اند. در این حالت، اشاره‌گر «next» را می‌توان به آدرس شروع هیپ بازنشانی کرد، به طوری که کل حافظه هیپ دوباره برای تخصیص‌ها در دسترس باشد.

### پیاده‌سازی

ما پیاده‌سازی خود را با تعریف یک زیر ماژول جدید 'allocator::bump' شروع می‌کنیم:

```rust
// in src/allocator.rs

pub mod bump;
```

محتوای زیر ماژول در فایل «src/allocator/bump.rs» قرار می‌گیرد که ما با محتوای زیر ایجاد می‌کنیم:

```rust
// in src/allocator/bump.rs

pub struct BumpAllocator {
    heap_start: usize,
    heap_end: usize,
    next: usize,
    allocations: usize,
}

impl BumpAllocator {
    /// Creates a new empty bump allocator.
    pub const fn new() -> Self {
        BumpAllocator {
            heap_start: 0,
            heap_end: 0,
            next: 0,
            allocations: 0,
        }
    }

    /// Initializes the bump allocator with the given heap bounds.
    ///
    /// This method is unsafe because the caller must ensure that the given
    /// memory range is unused. Also, this method must be called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.heap_start = heap_start;
        self.heap_end = heap_start + heap_size;
        self.next = heap_start;
    }
}
```

فیلدهای «heap_start» و «heap_end» مرز پایین و بالایی ناحیه حافظه هیپ را دنبال می‌کنند. فراخواننده باید مطمئن شود که این آدرس‌ها معتبر هستند، در غیر این صورت تخصیص‌دهنده حافظه نامعتبر را برمی‌گرداند. به همین دلیل، تابع «init» برای فراخوانی باید «unsafe» باشد.

هدف از فیلد «next» این است که همیشه به اولین بایت استفاده نشده هیپ اشاره کند، یعنی آدرس شروع تخصیص بعدی. در تابع 'init' روی 'heap_start' تنظیم شده است زیرا در ابتدا هیپ به صورت کامل استفاده نشده است. در هر تخصیص، این فیلد با اندازه تخصیص افزایش می‌یابد (_"bumped"_) تا اطمینان حاصل شود که منطقه حافظه مشابه را دو بار بر نمی‌گردانیم.

فیلد «allocations» یک شمارنده ساده برای تخصیص‌های فعال با هدف بازنشانی تخصیص‌دهنده پس از آزاد شدن آخرین تخصیص است. با 0 مقداردهی اولیه می‌شود.

ما تصمیم گرفتیم به جای اجرای مقداردهی اولیه به طور مستقیم در «new» یک تابع «init» جداگانه ایجاد کنیم تا رابط (interface) را با تخصیص‌دهنده ارائه شده توسط جعبه «linked_list_allocator» یکسان نگه داریم. به این ترتیب، تخصیص‌دهنده‌ها را می‌توان بدون تغییر دادن اضافی کد، سوئیچ کرد.

### پیاده‌سازی `GlobalAlloc`

همانطور که [در پست قبلی][global-alloc] توضیح داده شد، همه تخصیص‌دهنده‌های هیپ باید صفت ['GlobalAlloc'] را پیاده‌سازی کنند، که اینگونه تعریف می‌شود:

[global-alloc]: @/edition-2/posts/10-heap-allocation/index.md#the-allocator-interface
[`GlobalAlloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html

```rust
pub unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize
    ) -> *mut u8 { ... }
}
```

فقط متدهای 'alloc' و 'dealloc' مورد نیاز هستند، دو متد دیگر، پیاده‌سازی پیش‌فرض دارند و می توان آن‌ها را حذف کرد.

#### اولین تلاش برای پیاده‌سازی

بیایید سعی کنیم روش `alloc` را برای `BumpAllocator` خود پیاده‌سازی کنیم:

```rust
// in src/allocator/bump.rs

use alloc::alloc::{GlobalAlloc, Layout};

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // TODO alignment and bounds check
        let alloc_start = self.next;
        self.next = alloc_start + layout.size();
        self.allocations += 1;
        alloc_start as *mut u8
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        todo!();
    }
}
```

ابتدا از فیلد «next» به عنوان آدرس شروع برای تخصیص استفاده می‌کنیم. سپس فیلد «next» را به‌روزرسانی می‌کنیم تا به آدرس پایانی تخصیص، که آدرس استفاده نشده بعدی روی هیپ است، اشاره کند. قبل از برگرداندن آدرس شروع تخصیص به عنوان اشاره‌گر «*mut u8»، شمارنده «allocations» را یک واحد افزایش می‌دهیم.

توجه داشته باشید که ما هیچ‌گونه بررسی کرانه یا تنظیمات تراز را انجام نمی‌دهیم، بنابراین این پیاده‌سازی هنوز ایمن نیست. این خیلی مهم نیست زیرا به هر حال با خطای زیر شکست خورده و کامپایل نمی‌شود:

```
error[E0594]: cannot assign to `self.next` which is behind a `&` reference
  --> src/allocator/bump.rs:29:9
   |
29 |         self.next = alloc_start + layout.size();
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be written
```

(همان خطا برای خط `self.allocations += 1` نیز رخ می‌دهد. ما در اینجا برای اختصار آن را حذف کردیم.)

این خطا به این دلیل رخ می‌دهد که روش‌های ['alloc'] و ['dealloc'] مربوط به صفت 'GlobalAlloc'، فقط بر روی یک مرجع تغییرناپذیر '&self' کار می‌کنند، بنابراین به‌روزرسانی فیلدهای «next» و «allocations» امکان‌پذیر نیست. این مشکل‌ساز است زیرا به روز رسانی «next» در هر تخصیص، اصل اساسی یک تخصیص‌دهنده بامپ است.

[`alloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.alloc
[`dealloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.dealloc

#### صفت `GlobalAlloc` و تغییرپذیری

قبل از این‌که به یک راه حل ممکن برای این مشکل تغییرپذیری نگاه کنیم، بیایید سعی کنیم بفهمیم که چرا روش‌های صفت «GlobalAlloc» با آرگومان‌های «&self» تعریف می‌شوند: همانطور که [در پست قبلی][global-allocator] دیدیم، تخصیص‌دهنده هیپ سراسری با افزودن ویژگی «#[global_allocator]» به یک «static» که صفت «GlobalAlloc» را پیاده‌سازی می‌کند، تعریف می‌شود. متغیرهای استاتیک در راست، تغییرناپذیر هستند، بنابراین راهی برای فراخوانی متدی که «&mut self» را در تخصیص‌دهنده استاتیک می‌گیرد وجود ندارد. به همین دلیل، همه روش‌های «GlobalAlloc» فقط یک مرجع «&self» تغییرناپذیر می‌گیرند.

[global-allocator]:  @/edition-2/posts/10-heap-allocation/index.md#the-global-allocator-attribute

خوشبختانه راهی وجود دارد که می‌توان مرجع «&mut self» را از یک مرجع «&self» دریافت کرد: می‌توانیم با بسته‌بندی کردن تخصیص‌دهنده در یک اسپین‌لاک (spinlock) [`spin::Mutex`] از [تغییرپذیری داخلی] همگام‌سازی شده استفاده کنیم. این نوع یک روش `lock` ارائه می‌کند که [حذف متقابل] را انجام می‌دهد و بنابراین با خیال راحت یک مرجع `&self` را به مرجع «&mut self» تبدیل می‌کند. ما قبلاً از نوع wrapper چندین بار در هسته خود استفاده کردیم، به عنوان مثال برای [بافر متن VGA][vga-mutex].

[تغییرپذیری داخلی]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[vga-mutex]: @/edition-2/posts/03-vga-text-buffer/index.md#spinlocks
[`spin::Mutex`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html
[حذف متقابل]: https://en.wikipedia.org/wiki/Mutual_exclusion

#### یک نوع بسته‌بندی `Locked`

با کمک نوع wrapper «spin::Mutex» می‌توانیم صفت «GlobalAlloc» را برای تخصیص‌دهنده بامپ خود پیاده‌سازی کنیم. ترفند این است که این صفت را نه به طور مستقیم برای «BumpAllocator»، بلکه برای نوع «spin::Mutex<BumpAllocator>» بسته‌بندی شده پیاده‌سازی کنید:

```rust
unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {…}
```

متأسفانه، این هنوز کار نمی‌کند زیرا کامپایلر راست اجازه پیاده‌سازی صفت برای انواع تعریف شده در جعبه‌های دیگر را نمی‌دهد:

```
error[E0117]: only traits defined in the current crate can be implemented for arbitrary types
  --> src/allocator/bump.rs:28:1
   |
28 | unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^--------------------------
   | |                           |
   | |                           `spin::mutex::Mutex` is not defined in the current crate
   | impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

برای رفع این مشکل، باید نوع wrapper خودمان را در اطراف 'spin::Mutex' ایجاد کنیم:

```rust
// in src/allocator.rs

/// A wrapper around spin::Mutex to permit trait implementations.
pub struct Locked<A> {
    inner: spin::Mutex<A>,
}

impl<A> Locked<A> {
    pub const fn new(inner: A) -> Self {
        Locked {
            inner: spin::Mutex::new(inner),
        }
    }

    pub fn lock(&self) -> spin::MutexGuard<A> {
        self.inner.lock()
    }
}
```

این نوع، یک بسته‌بندی عمومی در اطراف یک «spin::Mutex<A>» است. هیچ محدودیتی برای نوع بسته‌بندی «A» اعمال نمی‌کند، بنابراین می‌توان از آن برای بسته‌بندی انواع مختلف، نه فقط تخصیص‌دهنده‌ها، استفاده کرد. این یک تابع سازنده ساده «new» ارائه می‌کند که یک مقدار مشخص را بسته‌بندی می‌کند. برای راحتی، یک تابع `lock` نیز ارائه می‌کند که «lock» را در «Mutex» بسته‌بندی شده فراخوانی می‌کند. از آن‌جایی که نوع `Locked` به اندازه کافی عمومی است که برای سایر پیاده‌سازی‌های تخصیص‌دهنده نیز مفید باشد، آن را در ماژول `allocator` والد قرار می‌دهیم.

#### پیاده‌سازی برای `Locked<BumpAllocator>`

نوع «Locked» در جعبه خود ما تعریف شده است (برخلاف «spin::Mutex»)، بنابراین می‌توانیم از آن برای پیاده‌سازی «GlobalAlloc» برای تخصیص‌دهنده بامپ خود استفاده کنیم. پیاده‌سازی کامل آن به صورت زیر است:

```rust
// in src/allocator/bump.rs

use super::{align_up, Locked};
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<BumpAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let mut bump = self.lock(); // get a mutable reference

        let alloc_start = align_up(bump.next, layout.align());
        let alloc_end = match alloc_start.checked_add(layout.size()) {
            Some(end) => end,
            None => return ptr::null_mut(),
        };

        if alloc_end > bump.heap_end {
            ptr::null_mut() // out of memory
        } else {
            bump.next = alloc_end;
            bump.allocations += 1;
            alloc_start as *mut u8
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        let mut bump = self.lock(); // get a mutable reference

        bump.allocations -= 1;
        if bump.allocations == 0 {
            bump.next = bump.heap_start;
        }
    }
}
```

اولین قدم برای «alloc» و «dealloc» فراخوانی متد [«Mutex::lock»] از طریق فیلد «inner» است تا یک مرجع تغییرپذیر به نوع تخصیص‌دهنده بسته‌بندی شده دریافت شود. این نمونه تا پایان متد قفل می‌ماند، به طوری که هیچ مسابقه داده‌ای (data race) در زمینه‌های چند نخی نمی‌تواند رخ دهد (به زودی پشتیبانی از نخ را اضافه خواهیم کرد).

[`Mutex::lock`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock

در مقایسه با نمونه اولیه قبلی، پیاده‌سازی «alloc» اکنون به الزامات هم‌ترازی احترام می‌گذارد و برای اطمینان از این‌که تخصیص‌ها در ناحیه حافظه هیپ باقی می‌مانند، یک بررسی کرانه‌ها (bounds check) انجام می‌دهد. اولین قدم گرد کردن آدرس «next» به تراز مشخص شده توسط آرگومان «Layout» است. کد تابع `align_up` بزودی نشان داده می شود. سپس اندازه تخصیص درخواستی را به `alloc_start` اضافه می‌کنیم تا آدرس پایانی تخصیص را بدست آوریم. برای جلوگیری از سرریز اعداد صحیح در تخصیص‌های بزرگ، از روش [`checked_add`] استفاده می‌کنیم. اگر سرریز اتفاق بیفتد یا اگر آدرس پایانی تخصیص از آدرس پایانی هیپ بزرگتر باشد، یک اشاره‌گر تهی برای سیگنال دادن به این‌که وضعیت out-of-memory رخ داده برمی‌گردانیم. در غیر این صورت، آدرس `next` را بروز می‌کنیم و شمارنده `allocations` را مانند قبل یک واحد افزایش می‌دهیم. در نهایت، آدرس «alloc_start» را که به اشاره‌گر «*mut u8» تبدیل شده است، برمی‌گردانیم.

[`checked_add`]: https://doc.rust-lang.org/std/primitive.usize.html#method.checked_add
[`Layout`]: https://doc.rust-lang.org/alloc/alloc/struct.Layout.html

تابع `dealloc` اشاره‌گر داده شده و آرگومان‌های `Layout` را نادیده می‌گیرد. در عوض، فقط شمارنده «allocations» را کاهش می‌دهد. اگر شمارنده دوباره به `0` رسید، به این معنی است که همه تخصیص ها دوباره آزاد شده‌اند. در این حالت، آدرس «next» را به آدرس «heap_start» بازنشانی می‌کند تا کل حافظه هیپ دوباره در دسترس باشد.

#### تراز آدرس

تابع `align_up` به اندازه کافی عمومی است که بتوانیم آن را در ماژول اصلی `allocator` قرار دهیم. یک پیاده‌سازی پایه به این صورت است:

```rust
// in src/allocator.rs

/// Align the given address `addr` upwards to alignment `align`.
fn align_up(addr: usize, align: usize) -> usize {
    let remainder = addr % align;
    if remainder == 0 {
        addr // addr already aligned
    } else {
        addr - remainder + align
    }
}
```

تابع ابتدا [باقی‌مانده] تقسیم «addr» بر «align» را محاسبه می‌کند. اگر باقی‌مانده «0» باشد، آدرس قبلاً با ترازِ داده شده، تراز شده است. در غیر این صورت، آدرس را با کم کردن باقی‌مانده (به طوری که باقی‌مانده جدید 0 باشد) و سپس اضافه کردن تراز (به طوری که آدرس از آدرس اصلی کوچکتر نشود)، تراز می‌کنیم.

[باقی‌مانده]: https://en.wikipedia.org/wiki/Euclidean_division

توجه داشته باشید که این کارآمدترین راه برای پیاده‌سازی این تابع نیست. پیاده‌سازی بسیار سریع‌تر به این صورت است:

```rust
/// Align the given address `addr` upwards to alignment `align`.
///
/// Requires that `align` is a power of two.
fn align_up(addr: usize, align: usize) -> usize {
    (addr + align - 1) & !(align - 1)
}
```

این متد از این استفاده می‌کند که صفت «GlobalAlloc» تضمین می‌کند که «align» همیشه توان دو است. این امکانِ ایجاد یک [bitmask] را برای تراز کردن آدرس به شیوه‌ای بسیار کارآمد را فراهم می‌کند. برای این‌که بفهمیم چگونه کار می‌کند، اجازه دهید مرحله به مرحله آن را از سمت راست شروع کنیم:

[bitmask]: https://en.wikipedia.org/wiki/Mask_(computing)

- از آن‌جایی که «align» توان دو است، [نمایش باینری] آن تنها یک مجموعه بیت دارد (مثلاً «0b000100000»). این بدان معناست که «align - 1» دارای تمام بیت‌های پایین‌تر است (مثلاً «0b00011111»).
- با ایجاد [bitwise `NOT`] از طریق عملگر «!»، عددی دریافت می‌کنیم که همه بیت‌ها به جز بیت‌های پایین‌تر از «align» تنظیم شده است (مثلاً «0b…111111111100000»).
- با انجام یک [bitwise `AND`] روی یک آدرس و `!(align - 1)`، آدرس _کاهشی_ را تراز می‌کنیم. این کار با پاک کردن تمام بیت‌هایی که پایین‌تر از «align» هستند انجام می‌شود.
- از آن‌جایی که می‌خواهیم به جای تراز کردن به سمت پایین به سمت بالا تراز کنیم، قبل از اجرای بیت «AND»، مقدار «addr» را با «align - 1» افزایش می‌دهیم. به این ترتیب، آدرس‌های از قبل تراز شده ثابت می‌مانند در حالی که آدرس‌های تراز نشده به مرز تراز بعدی گرد می‌شوند.

[نمایش باینری]: https://en.wikipedia.org/wiki/Binary_number#Representation
[bitwise `NOT`]: https://en.wikipedia.org/wiki/Bitwise_operation#NOT
[bitwise `AND`]: https://en.wikipedia.org/wiki/Bitwise_operation#AND

این‌که کدام نوع آن را انتخاب کنید به شما بستگی دارد. هر دو نتیجه‌ای یکسان را محاسبه می‌کنند، فقط از متدهای مختلف استفاده می‌کنند.

### استفاده کردن از آن

برای استفاده از تخصیص‌دهنده بامپ بجای جعبه «linked_list_allocator»، باید استاتیک «ALLOCATOR» را در «allocator.rs» بروزرسانی کنیم:

```rust
// in src/allocator.rs

use bump::BumpAllocator;

#[global_allocator]
static ALLOCATOR: Locked<BumpAllocator> = Locked::new(BumpAllocator::new());
```

در اینجا مهم است که «BumpAllocator::new» و «Locked::new» را به عنوان [تابع‌های `const`] تعریف کنیم. اگر آن‌ها توابع معمولی بودند، یک خطای کامپایل رخ می‌داد زیرا عبارت اولیه یک «static» باید در زمان کامپایل قابل ارزیابی باشد.

[تابع‌های `const`]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

ما نیازی به تغییر فراخوان `ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE)` در تابع «init_heap» نداریم زیرا تخصیص‌دهنده بامپ همان رابطی را ارائه می‌دهد که تخصیص‌دهنده ارائه شده توسط «linked_list_allocator» ارائه می‌دهد.

اکنون هسته از تخصیص‌دهنده بامپ ما استفاده می‌کند! همه چیز همچنان باید کار کند، از جمله [تست‌های `heap_allocation`] که در پست قبلی ایجاد کردیم:

[تست‌های `heap_allocation`]: @/edition-2/posts/10-heap-allocation/index.md#adding-a-test

```
> cargo test --test heap_allocation
[…]
Running 3 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
```

### بحث کردن

مزیت بزرگ تخصیص بامپ، بسیار سریع بودن آن است. در مقایسه با سایر طرح‌های تخصیص‌دهنده (به زیر مراجعه کنید) که باید به طور فعال به دنبال یک بلوک حافظه مناسب بگردند و وظایف ساماندهی مختلف را در «alloc» و «dealloc» انجام دهند، یک تخصیص‌دهنده بامپ را [می‌توان بهینه‌سازی کرد] به صورتی که تنها چند دستورالعمل اسمبلی باشد. این باعث می‌شود که تخصیص‌دهنده‌های بامپ برای بهینه‌سازی عملکرد تخصیص مفید باشند، به عنوان مثال هنگام ایجاد یک [کتابخانه مجازی DOM].

[bump downwards]: https://fitzgeraldnick.com/2019/11/01/always-bump-downwards.html
[کتابخانه مجازی DOM]: https://hacks.mozilla.org/2019/03/fast-bump-allocated-virtual-doms-with-rust-and-wasm/

در حالی که یک تخصیص‌دهنده به ندرت به عنوان تخصیص‌دهنده سراسری استفاده می‌شود، اصول تخصیص بامپ اغلب به شکل [تخصیص عرصه] (arena allocation) اعمال می‌شود، که اساسا تخصیص‌های تکی را با هم دسته‌بندی می‌کند تا عملکرد را بهبود ببخشد. نمونه‌ای برای تخصیص‌دهنده عرصه برای راست، جعبه ['toolshed'] است.

[تخصیص عرصه]: https://mgravell.github.io/Pipelines.Sockets.Unofficial/docs/arenas.html
[`toolshed`]: https://docs.rs/toolshed/0.8.1/toolshed/index.html

#### The Drawback of a Bump Allocator

محدودیت اصلی یک تخصیص‌دهنده بامپ این است که تنها پس از آزاد شدن همه تخصیص‌ها می‌تواند از حافظه اختصاص داده شده مجددا استفاده کند. این بدان معنی است که یک تخصیص طولانی مدت برای جلوگیری از استفاده مجدد از حافظه کافی است. ما می‌توانیم این را زمانی ببینیم که تغییری از تست «many_boxes» را اضافه کنیم:

```rust
// in tests/heap_allocation.rs

#[test_case]
fn many_boxes_long_lived() {
    let long_lived = Box::new(1); // new
    for i in 0..HEAP_SIZE {
        let x = Box::new(i);
        assert_eq!(*x, i);
    }
    assert_eq!(*long_lived, 1); // new
}
```

مانند تست «جعبه_های many_boxes»، این تست تعداد زیادی تخصیص ایجاد می‌کند تا در صورت عدم استفاده مجدد از حافظه آزاد شده توسط تخصیص‌دهنده، شکست out-of-memory ایجاد کند. علاوه‌بر این، تست تخصیص «long_lived» ایجاد می‌کند که برای کل اجرای حلقه زنده می‌ماند.

وقتی تست جدید خود را اجرا می‌کنیم، می‌بینیم که واقعاً شکست می‌خورد:

```
> cargo test --test heap_allocation
Running 4 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [failed]

Error: panicked at 'allocation error: Layout { size_: 8, align_: 8 }', src/lib.rs:86:5
```

بیایید سعی کنیم دلیل وقوع این شکست را با جزئیات بفهمیم: اول، تخصیص `long_lived` در ابتدای هیپ ایجاد می‌شود، در نتیجه شمارنده `allocations` را یک واحد افزایش می‌دهد. برای هر تکرار (iteration) حلقه، یک تخصیص کوتاه مدت ایجاد می‌شود. و مستقیماً قبل از شروع تکرار بعدی دوباره آزاد می‌شود. این بدان معناست که شمارنده «allocations» به طور موقت در ابتدای یک تکرار به 2 افزایش یافته و در پایان آن به 1 کاهش می‌یابد. مشکل اکنون این است که تخصیص‌دهنده بامپ تنها زمانی می‌تواند از حافظه مجدد استفاده کند که _همه_ تخصیص‌ها آزاد شده باشند، یعنی شمارنده «allocations» به 0 برسد. از آن‌جایی که این قبل از پایان حلقه اتفاق نمی‌افتد، هر تکرار حلقه، منطقه جدیدی از حافظه را اختصاص می‌دهد، که پس از چند بار تکرار منجر به خطای out-of-memory می‌شود.

#### اصلاح کردن تست؟

دو ترفند بالقوه وجود دارد که می‌توانیم از آن‌ها برای اصلاح تست تخصیص‌دهنده بامپ استفاده کنیم:

- می‌توانیم «dealloc» را بروزرسانی کنیم تا بررسی کنیم که آیا تخصیص آزاد شده آخرین تخصیصی است که «alloc» با مقایسه آدرس پایانی آن با اشاره‌گر «بعدی» بازگردانده است. در صورت مساوی بودن، می‌توانیم با خیال راحت «next» را به آدرس شروع تخصیص آزاد شده بازنشانی کنیم. به این ترتیب، هر تکرار حلقه از همان بلوک حافظه مجدداً استفاده می‌کند.
- می‌توانیم یک متد «alloc_back» اضافه کنیم که با استفاده از یک فیلد اضافی «next_back»، حافظه را از _انتهای_ هیپ تخصیص دهد. سپس می‌توانیم به‌صورت دستی از این متد تخصیص برای همه تخصیص‌های طولانی‌مدت استفاده کنیم و بدین ترتیب تخصیص‌های کوتاه‌مدت و طولانی‌مدت را در هیپ جدا کنیم. توجه داشته باشید که این جداسازی تنها در صورتی کار می‌کند که از قبل مشخص شده باشد که هر تخصیص چقدر طول می‌کشد. یکی دیگر از اشکالات این رویکرد این است که انجام دستی تخصیص‌ها دست و پا گیر و به طور بالقوه ناامن است.

در حالی که هر دوی این رویکردها برای اصلاح تست کار می‌کنند، اما راه حل کلی نیستند زیرا فقط در موارد بسیار خاص قادر به استفاده مجدد از حافظه هستند. سوال این است: آیا راه حل کلی برای استفاده مجدد از _تمام_ حافظه آزاد شده وجود دارد؟

#### استفاده مجدد از تمام حافظه آزاد شده؟

همانطور که [در پست قبلی][heap-intro] آموختیم، تخصیص‌ها می‌توانند به‌طور خودسرانه عمر طولانی داشته باشند و می‌توانند به ترتیب دلخواه آزاد شوند. این بدان معنی است که ما باید تعداد بالقوه نامحدودی از مناطق حافظه غیرمستمر و استفاده نشده را پیگیری کنیم، همانطور که در مثال زیر نشان داده شده است:

[heap-intro]: @/edition-2/posts/10-heap-allocation/index.md#dynamic-memory

![](allocation-fragmentation.svg)

تصویر بالا هیپ را در طول زمان نشان می‌دهد. در ابتدا، هیپ کامل استفاده نشده است و آدرس «next» برابر با «heap_start» است (خط 1). سپس اولین تخصیص رخ می‌دهد (خط 2). در خط 3، دومین بلوک حافظه تخصیص داده شده و اولین تخصیص آزاد می‌شود. تخصیص‌های بسیار بیشتری در خط 4 اضافه شده است. نیمی از آن‌ها بسیار کوتاه مدت هستند و در حال حاضر در خط 5 آزاد شده‌اند، جایی که تخصیص جدید دیگری نیز اضافه شده است.

خط 5 مشکل اساسی را نشان می‌دهد: ما در مجموع پنج منطقه حافظه استفاده نشده با اندازه‌های مختلف داریم، اما اشاره‌گر «next» فقط می‌تواند به ابتدای آخرین منطقه اشاره کند. در حالی که می‌توانیم آدرس‌های شروع و اندازه‌های دیگر مناطق حافظه استفاده نشده را در آرایه‌ای با اندازه 4 برای این مثال ذخیره کنیم، این یک راه حل کلی نیست زیرا می‌توانیم به راحتی یک مثال با 8، 16 یا 1000 منطقه حافظه استفاده نشده ایجاد کنیم.

معمولاً وقتی تعداد موارد بالقوه نامحدودی داریم، فقط می‌توانیم از یک مجموعه تخصیص داده شده هیپ استفاده کنیم. این واقعاً در مورد ما امکان‌پذیر نیست، زیرا تخصیص‌دهنده هیپ نمی‌تواند به خودش وابسته باشد (این امر باعث بازگشت بی‌پایان یا بن‌بست می‌شود). پس باید راه حل متفاوتی پیدا کنیم.

## تخصیص‌دهنده لیست پیوندی

یک ترفند رایج برای ردیابی تعداد دلخواه مناطق حافظه آزاد هنگام اجرای تخصیص‌دهنده‌ها، استفاده از خود این مناطق به عنوان حافظه پشتیبان است. زیرا می‌دانیم که مناطق هنوز به یک آدرس مجازی نگاشت شده و توسط یک فریم فیزیکی پشتیبانی می‌شوند، اما اطلاعات ذخیره شده دیگر مورد نیاز نیست. با ذخیره اطلاعات مربوط به منطقه آزاد شده در خود منطقه، می‌توانیم تعداد نامحدودی از مناطق آزاد شده را بدون نیاز به حافظه اضافی پیگیری کنیم.

رایج‌ترین رویکرد پیاده‌سازی، ساخت یک لیست پیوندی واحد در حافظه آزاد شده است که هر گره (node) یک منطقه حافظه آزاد شده است:

![](linked-list-allocation.svg)

هر گره لیست شامل دو فیلد است: اندازه ناحیه حافظه و یک اشاره‌گر به ناحیه حافظه استفاده نشده بعدی. با این رویکرد، ما فقط به یک اشاره‌گر به اولین منطقه استفاده نشده (به نام `head`) نیاز داریم تا همه مناطق استفاده نشده را مستقل از تعداد آن‌ها پیگیری کنیم. ساختار داده حاصل اغلب [_لیست رایگان_] (free list) نامیده می‌شود.

[_لیست رایگان_]: https://en.wikipedia.org/wiki/Free_list

همان‌طور که ممکن است از نام آن حدس بزنید، این تکنیکی است که جعبه «linked_list_allocator» از آن استفاده می‌کند. تخصیص‌دهنده‌هایی که از این تکنیک استفاده می‌کنند اغلب _pool allocators_ نیز نامیده می‌شوند.

### Implementation

In the following, we will create our own simple `LinkedListAllocator` type that uses the above approach for keeping track of freed memory regions. This part of the post isn't required for future posts, so you can skip the implementation details if you like.

#### The Allocator Type

We start by creating a private `ListNode` struct in a new `allocator::linked_list` submodule:

```rust
// in src/allocator.rs

pub mod linked_list;
```

```rust
// in src/allocator/linked_list.rs

struct ListNode {
    size: usize,
    next: Option<&'static mut ListNode>,
}
```

Like in the graphic, a list node has a `size` field and an optional pointer to the next node, represented by the `Option<&'static mut ListNode>` type. The `&'static mut` type semantically describes an [owned] object behind a pointer. Basically, it's a [`Box`] without a destructor that frees the object at the end of the scope.

[owned]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[`Box`]: https://doc.rust-lang.org/alloc/boxed/index.html

We implement the following set of methods for `ListNode`:

```rust
// in src/allocator/linked_list.rs

impl ListNode {
    const fn new(size: usize) -> Self {
        ListNode { size, next: None }
    }

    fn start_addr(&self) -> usize {
        self as *const Self as usize
    }

    fn end_addr(&self) -> usize {
        self.start_addr() + self.size
    }
}
```

The type has a simple constructor function named `new` and methods to calculate the start and end addresses of the represented region. We make the `new` function a [const function], which will be required later when constructing a static linked list allocator. Note that any use of mutable references in const functions (including setting the `next` field to `None`) is still unstable. In order to get it to compile, we need to add **`#![feature(const_mut_refs)]`** to the beginning of our `lib.rs`.

[const function]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

With the `ListNode` struct as building block, we can now create the `LinkedListAllocator` struct:

```rust
// in src/allocator/linked_list.rs

pub struct LinkedListAllocator {
    head: ListNode,
}

impl LinkedListAllocator {
    /// Creates an empty LinkedListAllocator.
    pub const fn new() -> Self {
        Self {
            head: ListNode::new(0),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.add_free_region(heap_start, heap_size);
    }

    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        todo!();
    }
}
```

The struct contains a `head` node that points to the first heap region. We are only interested in the value of the `next` pointer, so we set the `size` to 0 in the `ListNode::new` function. Making `head` a `ListNode` instead of just a `&'static mut ListNode` has the advantage that the implementation of the `alloc` method will be simpler.

Like for the bump allocator, the `new` function doesn't initialize the allocator with the heap bounds. In addition to maintaining API compatibility, the reason is that the initialization routine requires to write a node to the heap memory, which can only happen at runtime. The `new` function, however, needs to be a [`const` function] that can be evaluated at compile time, because it will be used for initializing the `ALLOCATOR` static. For this reason, we again provide a separate, non-constant `init` method.

[`const` function]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

The `init` method uses a `add_free_region` method, whose implementation will be shown in a moment. For now, we use the [`todo!`] macro to provide a placeholder implementation that always panics.

[`todo!`]: https://doc.rust-lang.org/core/macro.todo.html

#### The `add_free_region` Method

The `add_free_region` method provides the fundamental _push_ operation on the linked list. We currently only call this method from `init`, but it will also be the central method in our `dealloc` implementation. Remember, the `dealloc` method is called when an allocated memory region is freed again. To keep track of this freed memory region, we want to push it to the linked list.

The implementation of the `add_free_region` method looks like this:

```rust
// in src/allocator/linked_list.rs

use super::align_up;
use core::mem;

impl LinkedListAllocator {
    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        // ensure that the freed region is capable of holding ListNode
        assert_eq!(align_up(addr, mem::align_of::<ListNode>()), addr);
        assert!(size >= mem::size_of::<ListNode>());

        // create a new list node and append it at the start of the list
        let mut node = ListNode::new(size);
        node.next = self.head.next.take();
        let node_ptr = addr as *mut ListNode;
        node_ptr.write(node);
        self.head.next = Some(&mut *node_ptr)
    }
}
```

The method takes a memory region represented by an address and size as argument and adds it to the front of the list. First, it ensures that the given region has the necessary size and alignment for storing a `ListNode`. Then it creates the node and inserts it to the list through the following steps:

![](linked-list-allocator-push.svg)

Step 0 shows the state of the heap before `add_free_region` is called. In step 1, the method is called with the memory region marked as `freed` in the graphic. After the initial checks, the method creates a new `node` on its stack with the size of the freed region. It then uses the [`Option::take`] method to set the `next` pointer of the node to the current `head` pointer, thereby resetting the `head` pointer to `None`.

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

In step 2, the method writes the newly created `node` to the beginning of the freed memory region through the [`write`] method. It then points the `head` pointer to the new node. The resulting pointer structure looks a bit chaotic because the freed region is always inserted at the beginning of the list, but if we follow the pointers we see that each free region is still reachable from the `head` pointer.

[`write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

#### The `find_region` Method

The second fundamental operation on a linked list is finding an entry and removing it from the list. This is the central operation needed for implementing the `alloc` method. We implement the operation as a `find_region` method in the following way:

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Looks for a free region with the given size and alignment and removes
    /// it from the list.
    ///
    /// Returns a tuple of the list node and the start address of the allocation.
    fn find_region(&mut self, size: usize, align: usize)
        -> Option<(&'static mut ListNode, usize)>
    {
        // reference to current list node, updated for each iteration
        let mut current = &mut self.head;
        // look for a large enough memory region in linked list
        while let Some(ref mut region) = current.next {
            if let Ok(alloc_start) = Self::alloc_from_region(&region, size, align) {
                // region suitable for allocation -> remove node from list
                let next = region.next.take();
                let ret = Some((current.next.take().unwrap(), alloc_start));
                current.next = next;
                return ret;
            } else {
                // region not suitable -> continue with next region
                current = current.next.as_mut().unwrap();
            }
        }

        // no suitable region found
        None
    }
}
```

The method uses a `current` variable and a [`while let` loop] to iterate over the list elements. At the beginning, `current` is set to the (dummy) `head` node. On each iteration, it is then updated to the `next` field of the current node (in the `else` block). If the region is suitable for an allocation with the given size and alignment, the region is removed from the list and returned together with the `alloc_start` address.

[`while let` loop]: https://doc.rust-lang.org/reference/expressions/loop-expr.html#predicate-pattern-loops

When the `current.next` pointer becomes `None`, the loop exits. This means that we iterated over the whole list but found no region that is suitable for an allocation. In that case, we return `None`. The check whether a region is suitable is done by a `alloc_from_region` function, whose implementation will be shown in a moment.

Let's take a more detailed look at how a suitable region is removed from the list:

![](linked-list-allocator-remove-region.svg)

Step 0 shows the situation before any pointer adjustments. The `region` and `current` regions and the `region.next` and `current.next` pointers are marked in the graphic. In step 1, both the `region.next` and `current.next` pointers are reset to `None` by using the [`Option::take`] method. The original pointers are stored in local variables called `next` and `ret`.

In step 2, the `current.next` pointer is set to the local `next` pointer, which is the original `region.next` pointer. The effect is that `current` now directly points to the region after `region`, so that `region` is no longer element of the linked list. The function then returns the pointer to `region` stored in the local `ret` variable.

##### The `alloc_from_region` Function

The `alloc_from_region` function returns whether a region is suitable for an allocation with given size and alignment. It is defined like this:

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Try to use the given region for an allocation with given size and
    /// alignment.
    ///
    /// Returns the allocation start address on success.
    fn alloc_from_region(region: &ListNode, size: usize, align: usize)
        -> Result<usize, ()>
    {
        let alloc_start = align_up(region.start_addr(), align);
        let alloc_end = alloc_start.checked_add(size).ok_or(())?;

        if alloc_end > region.end_addr() {
            // region too small
            return Err(());
        }

        let excess_size = region.end_addr() - alloc_end;
        if excess_size > 0 && excess_size < mem::size_of::<ListNode>() {
            // rest of region too small to hold a ListNode (required because the
            // allocation splits the region in a used and a free part)
            return Err(());
        }

        // region suitable for allocation
        Ok(alloc_start)
    }
}
```

First, the function calculates the start and end address of a potential allocation, using the `align_up` function we defined earlier and the [`checked_add`] method. If an overflow occurs or if the end address is behind the end address of the region, the allocation doesn't fit in the region and we return an error.

The function performs a less obvious check after that. This check is necessary because most of the time an allocation does not fit a suitable region perfectly, so that a part of the region remains usable after the allocation. This part of the region must store its own `ListNode` after the allocation, so it must be large enough to do so. The check verifies exactly that: either the allocation fits perfectly (`excess_size == 0`) or the excess size is large enough to store a `ListNode`.

#### Implementing `GlobalAlloc`

With the fundamental operations provided by the `add_free_region` and `find_region` methods, we can now finally implement the `GlobalAlloc` trait. As with the bump allocator, we don't implement the trait directly for the `LinkedListAllocator`, but only for a wrapped `Locked<LinkedListAllocator>`. The [`Locked` wrapper] adds interior mutability through a spinlock, which allows us to modify the allocator instance even though the `alloc` and `dealloc` methods only take `&self` references.

[`Locked` wrapper]: @/edition-2/posts/11-allocator-designs/index.md#a-locked-wrapper-type

The implementation looks like this:

```rust
// in src/allocator/linked_list.rs

use super::Locked;
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<LinkedListAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // perform layout adjustments
        let (size, align) = LinkedListAllocator::size_align(layout);
        let mut allocator = self.lock();

        if let Some((region, alloc_start)) = allocator.find_region(size, align) {
            let alloc_end = alloc_start.checked_add(size).expect("overflow");
            let excess_size = region.end_addr() - alloc_end;
            if excess_size > 0 {
                allocator.add_free_region(alloc_end, excess_size);
            }
            alloc_start as *mut u8
        } else {
            ptr::null_mut()
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // perform layout adjustments
        let (size, _) = LinkedListAllocator::size_align(layout);

        self.lock().add_free_region(ptr as usize, size)
    }
}
```

Let's start with the `dealloc` method because it is simpler: First, it performs some layout adjustments, which we will explain in a moment, and retrieves a `&mut LinkedListAllocator` reference by calling the [`Mutex::lock`] function on the [`Locked` wrapper]. Then it calls the `add_free_region` function to add the deallocated region to the free list.

The `alloc` method is a bit more complex. It starts with the same layout adjustments and also calls the [`Mutex::lock`] function to receive a mutable allocator reference. Then it uses the `find_region` method to find a suitable memory region for the allocation and remove it from the list. If this doesn't succeed and `None` is returned, it returns `null_mut` to signal an error as there is no suitable memory region.

In the success case, the `find_region` method returns a tuple of the suitable region (no longer in the list) and the start address of the allocation. Using `alloc_start`, the allocation size, and the end address of the region, it calculates the end address of the allocation and the excess size again. If the excess size is not null, it calls `add_free_region` to add the excess size of the memory region back to the free list. Finally, it returns the `alloc_start` address casted as a `*mut u8` pointer.

#### Layout Adjustments

So what are these layout adjustments that we do at the beginning of both `alloc` and `dealloc`? They ensure that each allocated block is capable of storing a `ListNode`. This is important because the memory block is going to be deallocated at some point, where we want to write a `ListNode` to it. If the block is smaller than a `ListNode` or does not have the correct alignment, undefined behavior can occur.

The layout adjustments are performed by a `size_align` function, which is defined like this:

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Adjust the given layout so that the resulting allocated memory
    /// region is also capable of storing a `ListNode`.
    ///
    /// Returns the adjusted size and alignment as a (size, align) tuple.
    fn size_align(layout: Layout) -> (usize, usize) {
        let layout = layout
            .align_to(mem::align_of::<ListNode>())
            .expect("adjusting alignment failed")
            .pad_to_align();
        let size = layout.size().max(mem::size_of::<ListNode>());
        (size, layout.align())
    }
}
```

First, the function uses the [`align_to`] method on the passed [`Layout`] to increase the alignment to the alignment of a `ListNode` if necessary. It then uses the [`pad_to_align`] method to round up the size to a multiple of the alignment to ensure that the start address of the next memory block will have the correct alignment for storing a `ListNode` too.
In the second step it uses the [`max`] method to enforce a minimum allocation size of `mem::size_of::<ListNode>`. This way, the `dealloc` function can safely write a `ListNode` to the freed memory block.

[`align_to`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align_to
[`pad_to_align`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.pad_to_align
[`max`]: https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.max

### Using it

We can now update the `ALLOCATOR` static in the `allocator` module to use our new `LinkedListAllocator`:

```rust
// in src/allocator.rs

use linked_list::LinkedListAllocator;

#[global_allocator]
static ALLOCATOR: Locked<LinkedListAllocator> =
    Locked::new(LinkedListAllocator::new());
```

Since the `init` function behaves the same for the bump and linked list allocators, we don't need to modify the `init` call in `init_heap`.

When we now run our `heap_allocation` tests again, we see that all tests pass now, including the `many_boxes_long_lived` test that failed with the bump allocator:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

This shows that our linked list allocator is able to reuse freed memory for subsequent allocations.

### Discussion

In contrast to the bump allocator, the linked list allocator is much more suitable as a general purpose allocator, mainly because it is able to directly reuse freed memory. However, it also has some drawbacks. Some of them are only caused by our basic implementation, but there are also fundamental drawbacks of the allocator design itself.

#### Merging Freed Blocks

The main problem of our implementation is that it only splits the heap into smaller blocks, but never merges them back together. Consider this example:

![](linked-list-allocator-fragmentation-on-dealloc.svg)

In the first line, three allocations are created on the heap. Two of them are freed again in line 2 and the third is freed in line 3. Now the complete heap is unused again, but it is still split into four individual blocks. At this point, a large allocation might not be possible anymore because none of the four blocks is large enough. Over time, the process continues and the heap is split into smaller and smaller blocks. At some point, the heap is so fragmented that even normal sized allocations will fail.

To fix this problem, we need to merge adjacent freed blocks back together. For the above example, this would mean the following:

![](linked-list-allocator-merge-on-dealloc.svg)

Like before, two of the three allocations are freed in line `2`. Instead of keeping the fragmented heap, we now perform an additional step in line `2a` to merge the two rightmost blocks back together. In line `3`, the third allocation is freed (like before), resulting in a completely unused heap represented by three distinct blocks. In an additional merging step in line `3a` we then merge the three adjacent blocks back together.

The `linked_list_allocator` crate implements this merging strategy in the following way: Instead of inserting freed memory blocks at the beginning of the linked list on `deallocate`, it always keeps the list sorted by start address. This way, merging can be performed directly on the `deallocate` call by examining the addresses and sizes of the two neighbor blocks in the list. Of course, the deallocation operation is slower this way, but it prevents the heap fragmentation we saw above.

#### Performance

As we learned above, the bump allocator is extremely fast and can be optimized to just a few assembly operations. The linked list allocator performs much worse in this category. The problem is that an allocation request might need to traverse the complete linked list until it finds a suitable block.

Since the list length depends on the number of unused memory blocks, the performance can vary extremely for different programs. A program that only creates a couple of allocations will experience a relatively fast allocation performance. For a program that fragments the heap with many allocations, however, the allocation performance will be very bad because the linked list will be very long and mostly contain very small blocks.

It's worth noting that this performance issue isn't a problem caused by our basic implementation, but a fundamental problem of the linked list approach. Since allocation performance can be very important for kernel-level code, we explore a third allocator design in the following that trades improved performance for reduced memory utilization.

## Fixed-Size Block Allocator

In the following, we present an allocator design that uses fixed-size memory blocks for fulfilling allocation requests. This way, the allocator often returns blocks that are larger than needed for allocations, which results in wasted memory due to [internal fragmentation]. On the other hand, it drastically reduces the time required to find a suitable block (compared to the linked list allocator), resulting in much better allocation performance.

### Introduction

The idea behind a _fixed-size block allocator_ is the following: Instead of allocating exactly as much memory as requested, we define a small number of block sizes and round up each allocation to the next block size. For example, with block sizes of 16, 64, and 512 bytes, an allocation of 4 bytes would return a 16-byte block, an allocation of 48 bytes a 64-byte block, and an allocation of 128 bytes an 512-byte block.

Like the linked list allocator, we keep track of the unused memory by creating a linked list in the unused memory. However, instead of using a single list with different block sizes, we create a separate list for each size class. Each list then only stores blocks of a single size. For example, with block sizes 16, 64, and 512 there would be three separate linked lists in memory:

![](fixed-size-block-example.svg).

Instead of a single `head` pointer, we have the three head pointers `head_16`, `head_64`, and `head_512` that each point to the first unused block of the corresponding size. All nodes in a single list have the same size. For example, the list started by the `head_16` pointer only contains 16-byte blocks. This means that we no longer need to store the size in each list node since it is already specified by the name of the head pointer.

Since each element in a list has the same size, each list element is equally suitable for an allocation request. This means that we can very efficiently perform an allocation using the following steps:

- Round up the requested allocation size to the next block size. For example, when an allocation of 12 bytes is requested, we would choose the block size 16 in the above example.
- Retrieve the head pointer for the list, e.g. from an array. For block size 16, we need to use `head_16`.
- Remove the first block from the list and return it.

Most notably, we can always return the first element of the list and no longer need to traverse the full list. Thus, allocations are much faster than with the linked list allocator.

#### Block Sizes and Wasted Memory

Depending on the block sizes, we lose a lot of memory by rounding up. For example, when a 512-byte block is returned for a 128 byte allocation, three quarters of the allocated memory are unused. By defining reasonable block sizes, it is possible to limit the amount of wasted memory to some degree. For example, when using the powers of 2 (4, 8, 16, 32, 64, 128, …) as block sizes, we can limit the memory waste to half of the allocation size in the worst case and a quarter of the allocation size in the average case.

It is also common to optimize block sizes based on common allocation sizes in a program. For example, we could additionally add block size 24 to improve memory usage for programs that often perform allocations of 24 bytes. This way, the amount of wasted memory can be often reduced without losing the performance benefits.

#### Deallocation

Like allocation, deallocation is also very performant. It involves the following steps:

- Round up the freed allocation size to the next block size. This is required since the compiler only passes the requested allocation size to `dealloc`, not the size of the block that was returned by `alloc`. By using the same size-adjustment function in both `alloc` and `dealloc` we can make sure that we always free the correct amount of memory.
- Retrieve the head pointer for the list, e.g. from an array.
- Add the freed block to the front of the list by updating the head pointer.

Most notably, no traversal of the list is required for deallocation either. This means that the time required for a `dealloc` call stays the same regardless of the list length.

#### Fallback Allocator

Given that large allocations (>2KB) are often rare, especially in operating system kernels, it might make sense to fall back to a different allocator for these allocations. For example, we could fall back to a linked list allocator for allocations greater than 2048 bytes in order to reduce memory waste. Since only very few allocations of that size are expected, the linked list would stay small so that (de)allocations would be still reasonably fast.

#### Creating new Blocks

Above, we always assumed that there are always enough blocks of a specific size in the list to fulfill all allocation requests. However, at some point the linked list for a block size becomes empty. At this point, there are two ways how we can create new unused blocks of a specific size to fulfill an allocation request:

- Allocate a new block from the fallback allocator (if there is one).
- Split a larger block from a different list. This best works if block sizes are powers of two. For example, a 32-byte block can be split into two 16-byte blocks.

For our implementation, we will allocate new blocks from the fallback allocator since the implementation is much simpler.

### Implementation

Now that we know how a fixed-size block allocator works, we can start our implementation. We won't depend on the implementation of the linked list allocator created in the previous section, so you can follow this part even if you skipped the linked list allocator implementation.

#### List Node

We start our implementation by creating a `ListNode` type in a new `allocator::fixed_size_block` module:

```rust
// in src/allocator.rs

pub mod fixed_size_block;
```

```rust
// in src/allocator/fixed_size_block.rs

struct ListNode {
    next: Option<&'static mut ListNode>,
}
```

This type is similar to the `ListNode` type of our [linked list allocator implementation], with the difference that we don't have a second `size` field. The `size` field isn't needed because every block in a list has the same size with the fixed-size block allocator design.

[linked list allocator implementation]: #the-allocator-type

#### Block Sizes

Next, we define a constant `BLOCK_SIZES` slice with the block sizes used for our implementation:

```rust
// in src/allocator/fixed_size_block.rs

/// The block sizes to use.
///
/// The sizes must each be power of 2 because they are also used as
/// the block alignment (alignments must be always powers of 2).
const BLOCK_SIZES: &[usize] = &[8, 16, 32, 64, 128, 256, 512, 1024, 2048];
```

As block sizes, we use powers of 2 starting from 8 up to 2048. We don't define any block sizes smaller than 8 because each block must be capable of storing a 64-bit pointer to the next block when freed. For allocations greater than 2048 bytes we will fall back to a linked list allocator.

To simplify the implementation, we define that the size of a block is also its required alignment in memory. So a 16 byte block is always aligned on a 16-byte boundary and a 512 byte block is aligned on a 512-byte boundary. Since alignments always need to be powers of 2, this rules out any other block sizes. If we need block sizes that are not powers of 2 in the future, we can still adjust our implementation for this (e.g. by defining a second `BLOCK_ALIGNMENTS` array).

#### The Allocator Type

Using the `ListNode` type and the `BLOCK_SIZES` slice, we can now define our allocator type:

```rust
// in src/allocator/fixed_size_block.rs

pub struct FixedSizeBlockAllocator {
    list_heads: [Option<&'static mut ListNode>; BLOCK_SIZES.len()],
    fallback_allocator: linked_list_allocator::Heap,
}
```

The `list_heads` field is an array of `head` pointers, one for each block size. This is implemented by using the `len()` of the `BLOCK_SIZES` slice as the array length. As a fallback allocator for allocations larger than the largest block size we use the allocator provided by the `linked_list_allocator`. We could also used the `LinkedListAllocator` we implemented ourselves instead, but it has the disadvantage that it does not [merge freed blocks].

[merge freed blocks]: #merging-freed-blocks

For constructing a `FixedSizeBlockAllocator`, we provide the same `new` and `init` functions that we implemented for the other allocator types too:

```rust
// in src/allocator/fixed_size_block.rs

impl FixedSizeBlockAllocator {
    /// Creates an empty FixedSizeBlockAllocator.
    pub const fn new() -> Self {
        const EMPTY: Option<&'static mut ListNode> = None;
        FixedSizeBlockAllocator {
            list_heads: [EMPTY; BLOCK_SIZES.len()],
            fallback_allocator: linked_list_allocator::Heap::empty(),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.fallback_allocator.init(heap_start, heap_size);
    }
}
```

The `new` function just initializes the `list_heads` array with empty nodes and creates an [`empty`] linked list allocator as `fallback_allocator`. The `EMPTY` constant is needed because to tell the Rust compiler that we want to initialize the array with a constant value. Initializing the array directly as `[None; BLOCK_SIZES.len()]` does not work because then the compiler requires that `Option<&'static mut ListNode>` implements the `Copy` trait, which it does not. This is a current limitation of the Rust compiler, which might go away in the future.

[`empty`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.empty

If you haven't done so already for the `LinkedListAllocator` implementation, you also need to add **`#![feature(const_mut_refs)]`** to the beginning of your `lib.rs`. The reason is that any use of mutable reference types in const functions is still unstable, including the `Option<&'static mut ListNode>` array element type of the `list_heads` field (even if we set it to `None`).

The unsafe `init` function only calls the [`init`] function of the `fallback_allocator` without doing any additional initialization of the `list_heads` array. Instead, we will initialize the lists lazily on `alloc` and `dealloc` calls.

[`init`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.init

For convenience, we also create a private `fallback_alloc` method that allocates using the `fallback_allocator`:

```rust
// in src/allocator/fixed_size_block.rs

use alloc::alloc::Layout;
use core::ptr;

impl FixedSizeBlockAllocator {
    /// Allocates using the fallback allocator.
    fn fallback_alloc(&mut self, layout: Layout) -> *mut u8 {
        match self.fallback_allocator.allocate_first_fit(layout) {
            Ok(ptr) => ptr.as_ptr(),
            Err(_) => ptr::null_mut(),
        }
    }
}
```

Since the [`Heap`] type of the `linked_list_allocator` crate does not implement [`GlobalAlloc`] (as it's [not possible without locking]). Instead, it provides an [`allocate_first_fit`] method that has a slightly different interface. Instead of returning a `*mut u8` and using a null pointer to signal an error, it returns a `Result<NonNull<u8>, ()>`. The [`NonNull`] type is an abstraction for a raw pointer that is guaranteed to be not the null pointer. By mapping the `Ok` case to the [`NonNull::as_ptr`] method and the `Err` case to a null pointer, we can easily translate this back to a `*mut u8` type.

[`Heap`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html
[not possible without locking]: #globalalloc-and-mutability
[`allocate_first_fit`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.allocate_first_fit
[`NonNull`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html
[`NonNull::as_ptr`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html#method.as_ptr

#### Calculating the List Index

Before we implement the `GlobalAlloc` trait, we define a `list_index` helper function that returns the lowest possible block size for a given [`Layout`]:

```rust
// in src/allocator/fixed_size_block.rs

/// Choose an appropriate block size for the given layout.
///
/// Returns an index into the `BLOCK_SIZES` array.
fn list_index(layout: &Layout) -> Option<usize> {
    let required_block_size = layout.size().max(layout.align());
    BLOCK_SIZES.iter().position(|&s| s >= required_block_size)
}
```

The block must have at least the size and alignment required by the given `Layout`. Since we defined that the block size is also its alignment, this means that the `required_block_size` is the [maximum] of the layout's [`size()`] and [`align()`] attributes. To find the next-larger block in the `BLOCK_SIZES` slice, we first use the [`iter()`] method to get an iterator and then the [`position()`] method to find the index of the first block that is as least as large as the `required_block_size`.

[maximum]: https://doc.rust-lang.org/core/cmp/trait.Ord.html#method.max
[`size()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.size
[`align()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align
[`iter()`]: https://doc.rust-lang.org/std/primitive.slice.html#method.iter
[`position()`]:  https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.position

Note that we don't return the block size itself, but the index into the `BLOCK_SIZES` slice. The reason is that we want to use the returned index as an index into the `list_heads` array.

#### Implementing `GlobalAlloc`

The last step is to implement the `GlobalAlloc` trait:

```rust
// in src/allocator/fixed_size_block.rs

use super::Locked;
use alloc::alloc::GlobalAlloc;

unsafe impl GlobalAlloc for Locked<FixedSizeBlockAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        todo!();
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        todo!();
    }
}
```

Like for the other allocators, we don't implement the `GlobalAlloc` trait directly for our allocator type, but use the [`Locked` wrapper] to add synchronized interior mutability. Since the `alloc` and `dealloc` implementations are relatively large, we introduce them one by one in the following.

##### `alloc`

The implementation of the `alloc` method looks like this:

```rust
// in `impl` block in src/allocator/fixed_size_block.rs

unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            match allocator.list_heads[index].take() {
                Some(node) => {
                    allocator.list_heads[index] = node.next.take();
                    node as *mut ListNode as *mut u8
                }
                None => {
                    // no block exists in list => allocate new block
                    let block_size = BLOCK_SIZES[index];
                    // only works if all block sizes are a power of 2
                    let block_align = block_size;
                    let layout = Layout::from_size_align(block_size, block_align)
                        .unwrap();
                    allocator.fallback_alloc(layout)
                }
            }
        }
        None => allocator.fallback_alloc(layout),
    }
}
```

Let's go through it step by step:

First, we use the `Locked::lock` method to get a mutable reference to the wrapped allocator instance. Next, we call the `list_index` function we just defined to calculate the appropriate block size for the given layout and get the corresponding index into the `list_heads` array. If this index is `None`, no block size fits for the allocation, therefore we use the `fallback_allocator` using the `fallback_alloc` function.

If the list index is `Some`, we try to remove the first node in the corresponding list started by `list_heads[index]` using the [`Option::take`] method. If the list is not empty, we enter the `Some(node)` branch of the `match` statement, where we point the head pointer of the list to the successor of the popped `node` (by using [`take`][`Option::take`] again). Finally, we return the popped `node` pointer as a `*mut u8`.

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

If the list head is `None`, it indicates that the list of blocks is empty. This means that we need to construct a new block as [described above](#creating-new-blocks). For that, we first get the current block size from the `BLOCK_SIZES` slice and use it as both the size and the alignment for the new block. Then we create a new `Layout` from it and call the `fallback_alloc` method to perform the allocation. The reason for adjusting the layout and alignment is that the block will be added to the block list on deallocation.

#### `dealloc`

The implementation of the `dealloc` method looks like this:

```rust
// in src/allocator/fixed_size_block.rs

use core::{mem, ptr::NonNull};

// inside the `unsafe impl GlobalAlloc` block

unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            let new_node = ListNode {
                next: allocator.list_heads[index].take(),
            };
            // verify that block has size and alignment required for storing node
            assert!(mem::size_of::<ListNode>() <= BLOCK_SIZES[index]);
            assert!(mem::align_of::<ListNode>() <= BLOCK_SIZES[index]);
            let new_node_ptr = ptr as *mut ListNode;
            new_node_ptr.write(new_node);
            allocator.list_heads[index] = Some(&mut *new_node_ptr);
        }
        None => {
            let ptr = NonNull::new(ptr).unwrap();
            allocator.fallback_allocator.deallocate(ptr, layout);
        }
    }
}
```

Like in `alloc`, we first use the `lock` method to get a mutable allocator reference and then the `list_index` function to get the block list corresponding to the given `Layout`. If the index is `None`, no fitting block size exists in `BLOCK_SIZES`, which indicates that the allocation was created by the fallback allocator. Therefore we use its [`deallocate`][`Heap::deallocate`] to free the memory again. The method expects a [`NonNull`] instead of a `*mut u8`, so we need to convert the pointer first. (The `unwrap` call only fails when the pointer is null, which should never happen when the compiler calls `dealloc`.)

[`Heap::deallocate`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.deallocate

If `list_index` returns a block index, we need to add the freed memory block to the list. For that, we first create a new `ListNode` that points to the current list head (by using [`Option::take`] again). Before we write the new node into the freed memory block, we first assert that the current block size specified by `index` has the required size and alignment for storing a `ListNode`. Then we perform the write by converting the given `*mut u8` pointer to a `*mut ListNode` pointer and then calling the unsafe [`write`][`pointer::write`] method on it. The last step is to set the head pointer of the list, which is currently `None` since we called `take` on it, to our newly written `ListNode`. For that we convert the raw `new_node_ptr` to a mutable reference.

[`pointer::write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

There are a few things worth noting:

- We don't differentiate between blocks allocated from a block list and blocks allocated from the fallback allocator. This means that new blocks created in `alloc` are added to the block list on `dealloc`, thereby increasing the number of blocks of that size.
- The `alloc` method is the only place where new blocks are created in our implementation. This means that we initially start with empty block lists and only fill the lists lazily when allocations for that block size are performed.
- We don't need `unsafe` blocks in `alloc` and `dealloc`, even though we perform some `unsafe` operations. The reason is that Rust currently treats the complete body of unsafe functions as one large `unsafe` block. Since using explicit `unsafe` blocks has the advantage that it's obvious which operations are unsafe and which not, there is a [proposed RFC](https://github.com/rust-lang/rfcs/pull/2585) to change this behavior.

### Using it

To use our new `FixedSizeBlockAllocator`, we need to update the `ALLOCATOR` static in the `allocator` module:

```rust
// in src/allocator.rs

use fixed_size_block::FixedSizeBlockAllocator;

#[global_allocator]
static ALLOCATOR: Locked<FixedSizeBlockAllocator> = Locked::new(
    FixedSizeBlockAllocator::new());
```

Since the `init` function behaves the same for all allocators we implemented, we don't need to modify the `init` call in `init_heap`.

When we now run our `heap_allocation` tests again, all tests should still pass:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

Our new allocator seems to work!

### Discussion

While the fixed-size block approach has a much better performance than the linked list approach, it wastes up to half of the memory when using powers of 2 as block sizes. Whether this tradeoff is worth it heavily depends on the application type. For an operating system kernel, where performance is critical, the fixed-size block approach seems to be the better choice.

On the implementation side, there are various things that we could improve in our current implementation:

- Instead of only allocating blocks lazily using the fallback allocator, it might be better to pre-fill the lists to improve the performance of initial allocations.
- To simplify the implementation, we only allowed block sizes that are powers of 2 so that we could use them also as the block alignment. By storing (or calculating) the alignment in a different way, we could also allow arbitrary other block sizes. This way, we could add more block sizes, e.g. for common allocation sizes, in order to minimize the wasted memory.
- We currently only create new blocks, but never free them again. This results in fragmentation and might eventually result in allocation failure for large allocations. It might make sense to enforce a maximum list length for each block size. When the maximum length is reached, subsequent deallocations are freed using the fallback allocator instead of being added to the list.
- Instead of falling back to a linked list allocator, we could have a special allocator for allocations greater than 4KiB. The idea is to utilize [paging], which operates on 4KiB pages, to map a continuous block of virtual memory to non-continuous physical frames. This way, fragmentation of unused memory is no longer a problem for large allocations.
- With such a page allocator, it might make sense to add block sizes up to 4KiB and drop the linked list allocator completely. The main advantages of this would be reduced fragmentation and improved performance predictability, i.e. better worse-case performance.

[paging]: @/edition-2/posts/08-paging-introduction/index.md

It's important to note that the implementation improvements outlined above are only suggestions. Allocators used in operating system kernels are typically highly optimized to the specific workload of the kernel, which is only possible through extensive profiling.

### Variations

There are also many variations of the fixed-size block allocator design. Two popular examples are the _slab allocator_ and the _buddy allocator_, which are also used in popular kernels such as Linux. In the following, we give a short introduction to these two designs.

#### Slab Allocator

The idea behind a [slab allocator] is to use block sizes that directly correspond to selected types in the kernel. This way, allocations of those types fit a block size exactly and no memory is wasted. Sometimes, it might be even possible to preinitialize type instances in unused blocks to further improve performance.

[slab allocator]: https://en.wikipedia.org/wiki/Slab_allocation

Slab allocation is often combined with other allocators. For example, it can be used together with a fixed-size block allocator to further split an allocated block in order to reduce memory waste. It is also often used to implement an [object pool pattern] on top of a single large allocation.

[object pool pattern]: https://en.wikipedia.org/wiki/Object_pool_pattern

#### Buddy Allocator

Instead of using a linked list to manage freed blocks, the [buddy allocator] design uses a [binary tree] data structure together with power-of-2 block sizes. When a new block of a certain size is required, it splits a larger sized block into two halves, thereby creating two child nodes in the tree. Whenever a block is freed again, the neighbor block in the tree is analyzed. If the neighbor is also free, the two blocks are joined back together to a block of twice the size.

The advantage of this merge process is that [external fragmentation] is reduced so that small freed blocks can be reused for a large allocation. It also does not use a fallback allocator, so the performance is more predictable. The biggest drawback is that only power-of-2 block sizes are possible, which might result in a large amount of wasted memory due to [internal fragmentation]. For this reason, buddy allocators are often combined with a slab allocator to further split an allocated block into multiple smaller blocks.

[buddy allocator]: https://en.wikipedia.org/wiki/Buddy_memory_allocation
[binary tree]: https://en.wikipedia.org/wiki/Binary_tree
[external fragmentation]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation
[internal fragmentation]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation


## Summary

This post gave an overview over different allocator designs. We learned how to implement a basic [bump allocator], which hands out memory linearly by increasing a single `next` pointer. While bump allocation is very fast, it can only reuse memory after all allocations have been freed. For this reason, it is rarely used as a global allocator.

[bump allocator]: @/edition-2/posts/11-allocator-designs/index.md#bump-allocator

Next, we created a [linked list allocator] that uses the freed memory blocks itself to create a linked list, the so-called [free list]. This list makes it possible to store an arbitrary number of freed blocks of different sizes. While no memory waste occurs, the approach suffers from poor performance because an allocation request might require a complete traversal of the list. Our implementation also suffers from [external fragmentation] because it does not merge adjacent freed blocks back together.

[linked list allocator]: @/edition-2/posts/11-allocator-designs/index.md#linked-list-allocator
[free list]: https://en.wikipedia.org/wiki/Free_list

To fix the performance problems of the linked list approach, we created a [fixed-size block allocator] that predefines a fixed set of block sizes. For each block size, a separate [free list] exists so that allocations and deallocations only need to insert/pop at the front of the list and are thus very fast. Since each allocation is rounded up to the next larger block size, some memory is wasted due to [internal fragmentation].

[fixed-size block allocator]: @/edition-2/posts/11-allocator-designs/index.md#fixed-size-block-allocator

There are many more allocator designs with different tradeoffs. [Slab allocation] works well to optimize the allocation of common fixed-size structures, but is not applicable in all situations. [Buddy allocation] uses a binary tree to merge freed blocks back together, but wastes a large amount of memory because it only supports power-of-2 block sizes. It's also important to remember that each kernel implementation has a unique workload, so there is no "best" allocator design that fits all cases.

[Slab allocation]: @/edition-2/posts/11-allocator-designs/index.md#slab-allocator
[Buddy allocation]: @/edition-2/posts/11-allocator-designs/index.md#buddy-allocator


## What's next?

With this post, we conclude our memory management implementation for now. Next, we will start exploring [_multitasking_], starting with [_threads_]. In subsequent post we will then explore [_multiprocessing_], [_processes_], and cooperative multitasking in the form of [_async/await_].

[_multitasking_]: https://en.wikipedia.org/wiki/Computer_multitasking
[_threads_]: https://en.wikipedia.org/wiki/Thread_(computing)
[_processes_]: https://en.wikipedia.org/wiki/Process_(computing)
[_multiprocessing_]: https://en.wikipedia.org/wiki/Multiprocessing
[_async/await_]: https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html
