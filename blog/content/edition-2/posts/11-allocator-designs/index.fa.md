+++
title = "طرح‌های تخصیص‌دهنده"
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

در [پست قبلی] پشتیبانی اولیه برای تخصیص هیپ به هسته خود اضافه کردیم. برای آن، ما [یک ناحیه حافظه جدید][map-heap] در جدول‌های صفحه ایجاد کردیم و از [از کریت `linked_list_allocator`][use-alloc-crate] برای مدیریت آن حافظه استفاده کردیم. در حالی که ما اکنون یک هیپ داریم که به درستی کار می‌کند، بیشتر کار را به کریت تخصیص‌دهنده واگذار کردیم بدون اینکه سعی کنیم بفهمیم چگونه کار می‌کند.

[پست قبلی]: @/edition-2/posts/10-heap-allocation/index.md
[map-heap]: @/edition-2/posts/10-heap-allocation/index.md#creating-a-kernel-heap
[use-alloc-crate]: @/edition-2/posts/10-heap-allocation/index.md#using-an-allocator-crate

در این پست، ما نشان خواهیم داد که چگونه به جای تکیه بر یک کریت تخصیص‌دهنده موجود، تخصیص‌دهنده هیپ خود را از ابتدا ایجاد کنیم. ما در مورد طرح‌های تخصیص‌دهنده مختلف، از جمله یک _تخصیص‌دهنده بامپ_ ساده و یک _تخصیص‌دهنده بلوک با اندازه ثابت_ بحث خواهیم کرد، و از این دانش برای پیاده‌سازی یک تخصیص‌دهنده با عملکرد بهبودیافته (در مقایسه با کریت `linked_list_allocator`) استفاده خواهیم کرد.

### اهداف طراحی

مسئولیت تخصیص‌دهنده، مدیریت حافظه هیپ موجود است. باید حافظه استفاده نشده را در فراخوانی‌های `alloc` برگرداند و حافظه آزاد شده توسط `dealloc` را پیگیری کند تا بتوان دوباره از آن استفاده کرد. مهمتر از همه، هرگز نباید حافظه‌ای را که در حال حاضر در جای دیگری استفاده می‌شود را در اختیار دیگران قرار دهد زیرا این امر باعث رفتار نامشخص می‌شود.

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

ایده پشت تخصیص‌دهنده بامپ تخصیص خطی حافظه با افزایش (یا همان _"bumping"_) متغیر `next` است که به ابتدای حافظه استفاده نشده اشاره می‌کند. در ابتدا، `next` برابر با آدرس شروع هیپ است. در هر تخصیص، `next` با تخصیص افزایش می‌یابد به طوری که همیشه به مرز بین حافظه استفاده شده و استفاده نشده اشاره می‌کند:

![The heap memory area at three points in time:
 1: A single allocation exists at the start of the heap; the `next` pointer points to its end
 2: A second allocation was added right after the first; the `next` pointer points to the end of the second allocation
 3: A third allocation was added right after the second one; the `next pointer points to the end of the third allocation](bump-allocation.svg)
 
اشاره‌گر `next` فقط در یک جهت حرکت می‌کند و بنابراین هرگز یک ناحیه حافظه را دو بار به شما تحویل نمی‌دهد. هنگامی که به انتهای هیپ می‌رسد، دیگر حافظه قابل تخصیص نیست و در نتیجه در تخصیص بعدی خطای خارج از حافظه (out-of-memory) رخ می‌دهد.

یک تخصیص‌دهنده بامپ اغلب با یک شمارنده تخصیص پیاده‌سازی می‌شود که در هر فراخوانی `alloc` یک واحد افزایش می‌یابد و در هر فراخوانی `dealloc` یک واحد کاهش می‌یابد. هنگامی که شمارنده تخصیص به صفر می‌رسد به این معنی است که همه تخصیص‌ها در پشته تخصیص داده شده‌اند. در این حالت، اشاره‌گر `next` را می‌توان به آدرس شروع هیپ بازنشانی کرد، به طوری که کل حافظه هیپ دوباره برای تخصیص‌ها در دسترس باشد.

### پیاده‌سازی

ما پیاده‌سازی خود را با تعریف یک زیر ماژول جدید `allocator::bump` شروع می‌کنیم:

```rust
// in src/allocator.rs

pub mod bump;
```

محتوای زیر ماژول در فایل `src/allocator/bump.rs` قرار می‌گیرد که ما با محتوای زیر ایجاد می‌کنیم:

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

فیلدهای `heap_start` و `heap_end` مرز پایین و بالایی ناحیه حافظه هیپ را دنبال می‌کنند. فراخواننده باید مطمئن شود که این آدرس‌ها معتبر هستند، در غیر این صورت تخصیص‌دهنده حافظه نامعتبر را برمی‌گرداند. به همین دلیل، تابع `init` برای فراخوانی باید `unsafe` باشد.

هدف از فیلد `next` این است که همیشه به اولین بایت استفاده نشده هیپ اشاره کند، یعنی آدرس شروع تخصیص بعدی. در تابع `init` روی `heap_start` تنظیم شده است زیرا در ابتدا هیپ به صورت کامل استفاده نشده است. در هر تخصیص، این فیلد با اندازه تخصیص افزایش می‌یابد (_"bumped"_) تا اطمینان حاصل شود که ناحیه حافظه مشابه را دو بار بر نمی‌گردانیم.

فیلد `allocations` یک شمارنده ساده برای تخصیص‌های فعال با هدف بازنشانی تخصیص‌دهنده پس از آزاد شدن آخرین تخصیص است. با 0 مقداردهی اولیه می‌شود.

ما تصمیم گرفتیم به جای اجرای مقداردهی اولیه به طور مستقیم در `new` یک تابع `init` جداگانه ایجاد کنیم تا رابط (interface) را با تخصیص‌دهنده ارائه شده توسط کریت `linked_list_allocator` یکسان نگه داریم. به این ترتیب، تخصیص‌دهنده‌ها را می‌توان بدون تغییر دادن اضافی کد، سوئیچ کرد.

### پیاده‌سازی `GlobalAlloc`

همانطور که [در پست قبلی][global-alloc] توضیح داده شد، همه تخصیص‌دهنده‌های هیپ باید صفت [`GlobalAlloc`] را پیاده‌سازی کنند، که اینگونه تعریف می‌شود:

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

فقط متدهای `alloc` و `dealloc` مورد نیاز هستند، دو متد دیگر، پیاده‌سازی پیش‌فرض دارند و می توان آن‌ها را حذف کرد.

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

ابتدا از فیلد `next` به عنوان آدرس شروع برای تخصیص استفاده می‌کنیم. سپس فیلد `next` را به‌روزرسانی می‌کنیم تا به آدرس پایانی تخصیص، که آدرس استفاده نشده بعدی روی هیپ است، اشاره کند. قبل از برگرداندن آدرس شروع تخصیص به عنوان اشاره‌گر `*mut u8`، شمارنده `allocations` را یک واحد افزایش می‌دهیم.

توجه داشته باشید که ما هیچ‌گونه بررسی کرانه یا تنظیمات تراز را انجام نمی‌دهیم، بنابراین این پیاده‌سازی هنوز ایمن نیست. این خیلی مهم نیست زیرا به هر حال با خطای زیر شکست خورده و کامپایل نمی‌شود:

```
error[E0594]: cannot assign to `self.next` which is behind a `&` reference
  --> src/allocator/bump.rs:29:9
   |
29 |         self.next = alloc_start + layout.size();
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be written
```

(همان خطا برای خط `self.allocations += 1` نیز رخ می‌دهد. ما در اینجا برای اختصار آن را حذف کردیم.)

این خطا به این دلیل رخ می‌دهد که روش‌های [`alloc`] و [`dealloc`] مربوط به صفت `GlobalAlloc`، فقط بر روی یک مرجع تغییرناپذیر `&self` کار می‌کنند، بنابراین به‌روزرسانی فیلدهای `next` و `allocations` امکان‌پذیر نیست. این مشکل‌ساز است زیرا به روز رسانی `next` در هر تخصیص، اصل اساسی یک تخصیص‌دهنده بامپ است.

[`alloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.alloc
[`dealloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.dealloc

#### صفت `GlobalAlloc` و تغییرپذیری

قبل از این‌که به یک راه حل ممکن برای این مشکل تغییرپذیری نگاه کنیم، بیایید سعی کنیم بفهمیم که چرا روش‌های صفت `GlobalAlloc` با آرگومان‌های `&self` تعریف می‌شوند: همانطور که [در پست قبلی][global-allocator] دیدیم، تخصیص‌دهنده هیپ سراسری با افزودن ویژگی `#[global_allocator]` به یک `static` که صفت `GlobalAlloc` را پیاده‌سازی می‌کند، تعریف می‌شود. متغیرهای استاتیک در راست، تغییرناپذیر هستند، بنابراین راهی برای فراخوانی متدی که `&mut self` را در تخصیص‌دهنده استاتیک می‌گیرد وجود ندارد. به همین دلیل، همه روش‌های `GlobalAlloc` فقط یک مرجع `&self` تغییرناپذیر می‌گیرند.

[global-allocator]:  @/edition-2/posts/10-heap-allocation/index.md#the-global-allocator-attribute

خوشبختانه راهی وجود دارد که می‌توان مرجع `&mut self` را از یک مرجع `&self` دریافت کرد: می‌توانیم با بسته‌بندی کردن تخصیص‌دهنده در یک اسپین‌لاک (spinlock) [`spin::Mutex`] از [تغییرپذیری داخلی] همگام‌سازی شده استفاده کنیم. این نوع یک روش `lock` ارائه می‌کند که [حذف متقابل] را انجام می‌دهد و بنابراین با خیال راحت یک مرجع `&self` را به مرجع `&mut self` تبدیل می‌کند. ما قبلاً از نوع wrapper چندین بار در هسته خود استفاده کردیم، به عنوان مثال برای [بافر متن VGA][vga-mutex].

[تغییرپذیری داخلی]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[vga-mutex]: @/edition-2/posts/03-vga-text-buffer/index.md#spinlocks
[`spin::Mutex`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html
[حذف متقابل]: https://en.wikipedia.org/wiki/Mutual_exclusion

#### یک نوع بسته‌بندی `Locked`

با کمک نوع wrapper `spin::Mutex` می‌توانیم صفت `GlobalAlloc` را برای تخصیص‌دهنده بامپ خود پیاده‌سازی کنیم. ترفند این است که این صفت را نه به طور مستقیم برای `BumpAllocator`، بلکه برای نوع `spin::Mutex<BumpAllocator>` بسته‌بندی شده پیاده‌سازی کنید:

```rust
unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {…}
```

متأسفانه، این هنوز کار نمی‌کند زیرا کامپایلر راست اجازه پیاده‌سازی صفت برای انواع تعریف شده در کریت‌های دیگر را نمی‌دهد:

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

برای رفع این مشکل، باید نوع wrapper خودمان را در اطراف `spin::Mutex` ایجاد کنیم:

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

این نوع، یک بسته‌بندی عمومی در اطراف یک `spin::Mutex<A>` است. هیچ محدودیتی برای نوع بسته‌بندی `A` اعمال نمی‌کند، بنابراین می‌توان از آن برای بسته‌بندی انواع مختلف، نه فقط تخصیص‌دهنده‌ها، استفاده کرد. این یک تابع سازنده ساده `new` ارائه می‌کند که یک مقدار مشخص را بسته‌بندی می‌کند. برای راحتی، یک تابع `lock` نیز ارائه می‌کند که `lock` را در `Mutex` بسته‌بندی شده فراخوانی می‌کند. از آن‌جایی که نوع `Locked` به اندازه کافی عمومی است که برای سایر پیاده‌سازی‌های تخصیص‌دهنده نیز مفید باشد، آن را در ماژول `allocator` والد قرار می‌دهیم.

#### پیاده‌سازی برای `Locked<BumpAllocator>`

نوع `Locked` در کریت خود ما تعریف شده است (برخلاف `spin::Mutex`)، بنابراین می‌توانیم از آن برای پیاده‌سازی `GlobalAlloc` برای تخصیص‌دهنده بامپ خود استفاده کنیم. پیاده‌سازی کامل آن به صورت زیر است:

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

اولین قدم برای `alloc` و `dealloc` فراخوانی متد [`Mutex::lock`] از طریق فیلد `inner` است تا یک مرجع تغییرپذیر به نوع تخصیص‌دهنده بسته‌بندی شده دریافت شود. این نمونه تا پایان متد قفل می‌ماند، به طوری که هیچ مسابقه داده‌ای (data race) در زمینه‌های چند نخی نمی‌تواند رخ دهد (به زودی پشتیبانی از نخ را اضافه خواهیم کرد).

[`Mutex::lock`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock

در مقایسه با نمونه اولیه قبلی، پیاده‌سازی `alloc` اکنون به الزامات هم‌ترازی احترام می‌گذارد و برای اطمینان از این‌که تخصیص‌ها در ناحیه حافظه هیپ باقی می‌مانند، یک بررسی کرانه‌ها (bounds check) انجام می‌دهد. اولین قدم گرد کردن آدرس `next` به تراز مشخص شده توسط آرگومان `Layout` است. کد تابع `align_up` به‌زودی نشان داده می شود. سپس اندازه تخصیص درخواستی را به `alloc_start` اضافه می‌کنیم تا آدرس پایانی تخصیص را بدست آوریم. برای جلوگیری از سرریز اعداد صحیح در تخصیص‌های بزرگ، از روش [`checked_add`] استفاده می‌کنیم. اگر سرریز اتفاق بیفتد یا اگر آدرس پایانی تخصیص از آدرس پایانی هیپ بزرگتر باشد، یک اشاره‌گر تهی برای سیگنال دادن به این‌که وضعیت out-of-memory رخ داده برمی‌گردانیم. در غیر این صورت، آدرس `next` را بروز می‌کنیم و شمارنده `allocations` را مانند قبل یک واحد افزایش می‌دهیم. در نهایت، آدرس `alloc_start` را که به اشاره‌گر `*mut u8` تبدیل شده است، برمی‌گردانیم.

[`checked_add`]: https://doc.rust-lang.org/std/primitive.usize.html#method.checked_add
[`Layout`]: https://doc.rust-lang.org/alloc/alloc/struct.Layout.html

تابع `dealloc` اشاره‌گر داده شده و آرگومان‌های `Layout` را نادیده می‌گیرد. در عوض، فقط شمارنده `allocations` را کاهش می‌دهد. اگر شمارنده دوباره به `0` رسید، به این معنی است که همه تخصیص ها دوباره آزاد شده‌اند. در این حالت، آدرس `next` را به آدرس `heap_start` بازنشانی می‌کند تا کل حافظه هیپ دوباره در دسترس باشد.

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

تابع ابتدا [باقی‌مانده] تقسیم `addr` بر `align` را محاسبه می‌کند. اگر باقی‌مانده `0` باشد، آدرس قبلاً با ترازِ داده شده، تراز شده است. در غیر این صورت، آدرس را با کم کردن باقی‌مانده (به طوری که باقی‌مانده جدید 0 باشد) و سپس اضافه کردن تراز (به طوری که آدرس از آدرس اصلی کوچکتر نشود)، تراز می‌کنیم.

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

این متد از این استفاده می‌کند که صفت `GlobalAlloc` تضمین می‌کند که `align` همیشه توان دو است. این امکانِ ایجاد یک [bitmask] را برای تراز کردن آدرس به شیوه‌ای بسیار کارآمد را فراهم می‌کند. برای این‌که بفهمیم چگونه کار می‌کند، اجازه دهید مرحله به مرحله آن را از سمت راست شروع کنیم:

[bitmask]: https://en.wikipedia.org/wiki/Mask_(computing)

- از آن‌جایی که `align` توان دو است، [نمایش باینری] آن تنها یک مجموعه بیت دارد (مثلاً `0b000100000`). این بدان معناست که `align - 1` دارای تمام بیت‌های پایین‌تر است (مثلاً `0b00011111`).
- با ایجاد [bitwise `NOT`] از طریق عملگر `!`، عددی دریافت می‌کنیم که همه بیت‌ها به جز بیت‌های پایین‌تر از `align` تنظیم شده است (مثلاً `0b…111111111100000`).
- با انجام یک [bitwise `AND`] روی یک آدرس و `!(align - 1)`، آدرس _کاهشی_ را تراز می‌کنیم. این کار با پاک کردن تمام بیت‌هایی که پایین‌تر از `align` هستند انجام می‌شود.
- از آن‌جایی که می‌خواهیم به جای تراز کردن به سمت پایین به سمت بالا تراز کنیم، قبل از اجرای بیت `AND`، مقدار `addr` را با `align - 1` افزایش می‌دهیم. به این ترتیب، آدرس‌های از قبل تراز شده ثابت می‌مانند در حالی که آدرس‌های تراز نشده به مرز تراز بعدی گرد می‌شوند.

[نمایش باینری]: https://en.wikipedia.org/wiki/Binary_number#Representation
[bitwise `NOT`]: https://en.wikipedia.org/wiki/Bitwise_operation#NOT
[bitwise `AND`]: https://en.wikipedia.org/wiki/Bitwise_operation#AND

این‌که کدام نوع آن را انتخاب کنید به شما بستگی دارد. هر دو نتیجه‌ای یکسان را محاسبه می‌کنند، فقط از متدهای مختلف استفاده می‌کنند.

### استفاده کردن از آن

برای استفاده از تخصیص‌دهنده بامپ بجای کریت `linked_list_allocator`، باید استاتیک `ALLOCATOR` را در `allocator.rs` به‌روزرسانی کنیم:

```rust
// in src/allocator.rs

use bump::BumpAllocator;

#[global_allocator]
static ALLOCATOR: Locked<BumpAllocator> = Locked::new(BumpAllocator::new());
```

در اینجا مهم است که `BumpAllocator::new` و `Locked::new` را به عنوان [تابع‌های `const`] تعریف کنیم. اگر آن‌ها توابع معمولی بودند، یک خطای کامپایل رخ می‌داد زیرا عبارت اولیه یک `static` باید در زمان کامپایل قابل ارزیابی باشد.

[تابع‌های `const`]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

ما نیازی به تغییر فراخوان `ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE)` در تابع `init_heap` نداریم زیرا تخصیص‌دهنده بامپ همان رابطی را ارائه می‌دهد که تخصیص‌دهنده ارائه شده توسط `linked_list_allocator` ارائه می‌دهد.

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

مزیت بزرگ تخصیص بامپ، بسیار سریع بودن آن است. در مقایسه با سایر طرح‌های تخصیص‌دهنده (به زیر مراجعه کنید) که باید به طور فعال به دنبال یک بلوک حافظه مناسب بگردند و وظایف ساماندهی مختلف را در `alloc` و `dealloc` انجام دهند، یک تخصیص‌دهنده بامپ را [می‌توان بهینه‌سازی کرد] به صورتی که تنها چند دستورالعمل اسمبلی باشد. این باعث می‌شود که تخصیص‌دهنده‌های بامپ برای بهینه‌سازی عملکرد تخصیص مفید باشند، به عنوان مثال هنگام ایجاد یک [کتابخانه مجازی DOM].

[bump downwards]: https://fitzgeraldnick.com/2019/11/01/always-bump-downwards.html
[کتابخانه مجازی DOM]: https://hacks.mozilla.org/2019/03/fast-bump-allocated-virtual-doms-with-rust-and-wasm/

در حالی که یک تخصیص‌دهنده به ندرت به عنوان تخصیص‌دهنده سراسری استفاده می‌شود، اصول تخصیص بامپ اغلب به شکل [تخصیص عرصه] (arena allocation) اعمال می‌شود، که اساسا تخصیص‌های تکی را با هم دسته‌بندی می‌کند تا عملکرد را بهبود ببخشد. نمونه‌ای برای تخصیص‌دهنده عرصه برای راست، کریت [`toolshed`] است.

[تخصیص عرصه]: https://mgravell.github.io/Pipelines.Sockets.Unofficial/docs/arenas.html
[`toolshed`]: https://docs.rs/toolshed/0.8.1/toolshed/index.html

#### The Drawback of a Bump Allocator

محدودیت اصلی یک تخصیص‌دهنده بامپ این است که تنها پس از آزاد شدن همه تخصیص‌ها می‌تواند از حافظه اختصاص داده شده مجددا استفاده کند. این بدان معنی است که یک تخصیص طولانی مدت برای جلوگیری از استفاده مجدد از حافظه کافی است. ما می‌توانیم این را زمانی ببینیم که تغییری از تست `many_boxes` را اضافه کنیم:

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

مانند تست `کریت_های many_boxes`، این تست تعداد زیادی تخصیص ایجاد می‌کند تا در صورت عدم استفاده مجدد از حافظه آزاد شده توسط تخصیص‌دهنده، شکست out-of-memory ایجاد کند. علاوه‌بر این، تست تخصیص `long_lived` ایجاد می‌کند که برای کل اجرای حلقه زنده می‌ماند.

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

بیایید سعی کنیم دلیل وقوع این شکست را با جزئیات بفهمیم: اول، تخصیص `long_lived` در ابتدای هیپ ایجاد می‌شود، در نتیجه شمارنده `allocations` را یک واحد افزایش می‌دهد. برای هر تکرار (iteration) حلقه، یک تخصیص کوتاه مدت ایجاد می‌شود. و مستقیماً قبل از شروع تکرار بعدی دوباره آزاد می‌شود. این بدان معناست که شمارنده `allocations` به طور موقت در ابتدای یک تکرار به 2 افزایش یافته و در پایان آن به 1 کاهش می‌یابد. مشکل اکنون این است که تخصیص‌دهنده بامپ تنها زمانی می‌تواند از حافظه مجدد استفاده کند که _همه_ تخصیص‌ها آزاد شده باشند، یعنی شمارنده `allocations` به 0 برسد. از آن‌جایی که این قبل از پایان حلقه اتفاق نمی‌افتد، هر تکرار حلقه، ناحیه جدیدی از حافظه را اختصاص می‌دهد، که پس از چند بار تکرار منجر به خطای out-of-memory می‌شود.

#### اصلاح کردن تست؟

دو ترفند بالقوه وجود دارد که می‌توانیم از آن‌ها برای اصلاح تست تخصیص‌دهنده بامپ استفاده کنیم:

- می‌توانیم `dealloc` را به‌روزرسانی کنیم تا بررسی کنیم که آیا تخصیص آزاد شده آخرین تخصیصی است که `alloc` با مقایسه آدرس پایانی آن با اشاره‌گر `بعدی` بازگردانده است. در صورت مساوی بودن، می‌توانیم با خیال راحت `next` را به آدرس شروع تخصیص آزاد شده بازنشانی کنیم. به این ترتیب، هر تکرار حلقه از همان بلوک حافظه مجدداً استفاده می‌کند.
- می‌توانیم یک متد `alloc_back` اضافه کنیم که با استفاده از یک فیلد اضافی `next_back`، حافظه را از _انتهای_ هیپ تخصیص دهد. سپس می‌توانیم به‌صورت دستی از این متد تخصیص برای همه تخصیص‌های طولانی‌مدت استفاده کنیم و بدین ترتیب تخصیص‌های کوتاه‌مدت و طولانی‌مدت را در هیپ جدا کنیم. توجه داشته باشید که این جداسازی تنها در صورتی کار می‌کند که از قبل مشخص شده باشد که هر تخصیص چقدر طول می‌کشد. یکی دیگر از اشکالات این رویکرد این است که انجام دستی تخصیص‌ها دست و پا گیر و به طور بالقوه ناامن است.

در حالی که هر دوی این رویکردها برای اصلاح تست کار می‌کنند، اما راه حل کلی نیستند زیرا فقط در موارد بسیار خاص قادر به استفاده مجدد از حافظه هستند. سوال این است: آیا راه حل کلی برای استفاده مجدد از _تمام_ حافظه آزاد شده وجود دارد؟

#### استفاده مجدد از تمام حافظه آزاد شده؟

همانطور که [در پست قبلی][heap-intro] آموختیم، تخصیص‌ها می‌توانند به‌طور خودسرانه عمر طولانی داشته باشند و می‌توانند به ترتیب دلخواه آزاد شوند. این بدان معنی است که ما باید تعداد بالقوه نامحدودی از ناحیه‌های حافظه غیرمستمر و استفاده نشده را پیگیری کنیم، همانطور که در مثال زیر نشان داده شده است:

[heap-intro]: @/edition-2/posts/10-heap-allocation/index.md#dynamic-memory

![](allocation-fragmentation.svg)

تصویر بالا هیپ را در طول زمان نشان می‌دهد. در ابتدا، هیپ کامل استفاده نشده است و آدرس `next` برابر با `heap_start` است (خط 1). سپس اولین تخصیص رخ می‌دهد (خط 2). در خط 3، دومین بلوک حافظه تخصیص داده شده و اولین تخصیص آزاد می‌شود. تخصیص‌های بسیار بیشتری در خط 4 اضافه شده است. نیمی از آن‌ها بسیار کوتاه مدت هستند و در حال حاضر در خط 5 آزاد شده‌اند، جایی که تخصیص جدید دیگری نیز اضافه شده است.

خط 5 مشکل اساسی را نشان می‌دهد: ما در مجموع پنج ناحیه حافظه استفاده نشده با اندازه‌های مختلف داریم، اما اشاره‌گر `next` فقط می‌تواند به ابتدای آخرین ناحیه اشاره کند. در حالی که می‌توانیم آدرس‌های شروع و اندازه‌های دیگر ناحیه‌های حافظه استفاده نشده را در آرایه‌ای با اندازه 4 برای این مثال ذخیره کنیم، این یک راه حل کلی نیست زیرا می‌توانیم به راحتی یک مثال با 8، 16 یا 1000 ناحیه حافظه استفاده نشده ایجاد کنیم.

معمولاً وقتی تعداد موارد بالقوه نامحدودی داریم، فقط می‌توانیم از یک مجموعه تخصیص داده شده هیپ استفاده کنیم. این واقعاً در مورد ما امکان‌پذیر نیست، زیرا تخصیص‌دهنده هیپ نمی‌تواند به خودش وابسته باشد (این امر باعث بازگشت بی‌پایان یا بن‌بست می‌شود). پس باید راه حل متفاوتی پیدا کنیم.

## تخصیص‌دهنده لیست پیوندی

یک ترفند رایج برای ردیابی تعداد دلخواه ناحیه‌های حافظه آزاد هنگام اجرای تخصیص‌دهنده‌ها، استفاده از خود این ناحیه‌های به عنوان حافظه پشتیبان است. زیرا می‌دانیم که ناحیه‌های هنوز به یک آدرس مجازی نگاشت شده و توسط یک فریم فیزیکی پشتیبانی می‌شوند، اما اطلاعات ذخیره شده دیگر مورد نیاز نیست. با ذخیره اطلاعات مربوط به ناحیه آزاد شده در خود ناحیه، می‌توانیم تعداد نامحدودی از ناحیه‌های آزاد شده را بدون نیاز به حافظه اضافی پیگیری کنیم.

رایج‌ترین رویکرد پیاده‌سازی، ساخت یک لیست پیوندی واحد در حافظه آزاد شده است که هر گره (node) یک ناحیه حافظه آزاد شده است:

![](linked-list-allocation.svg)

هر گره لیست شامل دو فیلد است: اندازه ناحیه حافظه و یک اشاره‌گر به ناحیه حافظه استفاده نشده بعدی. با این رویکرد، ما فقط به یک اشاره‌گر به اولین ناحیه استفاده نشده (به نام `head`) نیاز داریم تا همه ناحیه‌های استفاده نشده را مستقل از تعداد آن‌ها پیگیری کنیم. ساختار داده حاصل اغلب [_لیست رایگان_] (free list) نامیده می‌شود.

[_لیست رایگان_]: https://en.wikipedia.org/wiki/Free_list

همان‌طور که ممکن است از نام آن حدس بزنید، این تکنیکی است که کریت `linked_list_allocator` از آن استفاده می‌کند. تخصیص‌دهنده‌هایی که از این تکنیک استفاده می‌کنند اغلب _pool allocators_ نیز نامیده می‌شوند.

### پیاده‌سازی

در ادامه، نوع ساده `LinkedListAllocator` خود را ایجاد خواهیم کرد که از رویکرد بالا برای پیگیری ناحیه‌های آزاد شده حافظه استفاده می‌کند. این قسمت از پست برای پست‌های بعدی مورد نیاز نیست، بنابراین در صورت تمایل می‌توانید جزئیات پیاده‌سازی را نادیده بگیرید.

#### نوع تخصیص‌دهنده

ما با ایجاد یک ساختمان `ListNode` خصوصی در یک زیر ماژول جدید `allocator::linked_list` شروع می‌کنیم:

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

مانند تصویر، یک لیست گره دارای یک فیلد `size` و یک اشاره‌گر اختیاری به گره بعدی است که با نوع `Option<&'static mut ListNode>` نمایش داده می‌شود. نوع `&'static mut` به صورت معنایی یک شیء [تعلق داده شده] را در پشت یک اشاره‌گر توصیف می‌کند. اساساً، این یک [`Box`] بدون مخرب است که شیء را در انتهای محدوده آزاد می‌کند.

[owned]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[`Box`]: https://doc.rust-lang.org/alloc/boxed/index.html

ما مجموعه متدهای زیر را برای `ListNode` پیاده‌سازی می‌کنیم:

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

این نوع دارای یک تابع سازنده ساده به نام `new` و متدهایی برای محاسبه آدرس‌های شروع و پایان ناحیه مربوطه است. ما تابع `new` را یک [تابع const] می‌کنیم، که بعداً هنگام ساخت یک تخصیص‌دهنده لیست پیوندی استاتیک مورد نیاز خواهد بود. توجه داشته باشید که هرگونه استفاده از مراجع قابل تغییر در توابع const (از جمله تنظیم فیلد `next` با مقدار `None`) همچنان ناپایدار است. برای این‌که بتوانیم آن را کامپایل کنیم، باید **`#![feature(const_mut_refs)]`** را به ابتدای `lib.rs` خود اضافه کنیم.

[تابع const]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

با ساختمان `ListNode` به عنوان بلوک سازنده، اکنون می‌توانیم ساختمان `LinkedListAllocator` را ایجاد کنیم:

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

ساختمان شامل یک گره `head` است که به اولین ناحیه هیپ اشاره می‌کند. ما فقط به مقدار اشاره‌گر `next` نیاز داریم، بنابراین `size` را در تابع `ListNode::new` روی 0 قرار می‌دهیم. ساختن `head` به عنوان یک `ListNode` به جای `&'static mut ListNode` این مزیت را دارد که پیاده‌سازی متد `alloc` ساده‌تر خواهد بود.

مانند تخصیص‌دهنده بامپ، تابع `new` تخصیص‌دهنده را با کران هیپ مقداردهی اولیه نمی‌کند. علاوه‌بر حفظ سازگاری API، دلیلش این است که روال مقداردهی اولیه نیاز به نوشتن یک گره در حافظه هیپ دارد که فقط در زمان اجرا می‌تواند اتفاق بیفتد. با این حال، تابع `new` باید یک تابع [`const`] باشد که بتوان آن را در زمان کامپایل ارزیابی کرد، زیرا برای مقداردهی اولیه استاتیک `ALLOCATOR` استفاده خواهد شد. به همین دلیل، ما دوباره یک متد `init` مجزا و غیر ثابت (non-constant) ارائه می‌کنیم.

[`const` function]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

متد `init` از متد `add_free_region` استفاده می‌کند که پیاده‌سازی آن به‌زودی نشان داده می‌شود. در حال حاضر، ما از ماکرو [`todo!`] برای ارائه یک نگهدارنده مکان که همیشه پنیک زده می‌کند استفاده می‌کنیم.

[`todo!`]: https://doc.rust-lang.org/core/macro.todo.html

#### متد `add_free_region`

متد `add_free_region` عملیات اساسی _پوش کردن_ را در لیست پیوند شده ارائه می‌کند. ما در حال حاضر فقط این متد را از `init` فراخوانی می‌کنیم، اما این متد در پیاده‌سازی `dealloc` ما نیز خواهد بود. به یاد داشته باشید، متد `dealloc` زمانی فراخوانی می‌شود که یک ناحیه حافظه اختصاص داده شده دوباره آزاد شود. برای پیگیری این ناحیه حافظه آزاد شده، می‌خواهیم آن را به لیست پیوندی منتقل کنیم.

پیاده‌سازی متد `add_free_region` به شکل زیر است:

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

این متد یک ناحیه حافظه که با آدرس و اندازه نشان داده شده است را به عنوان آرگومان می‌گیرد و آن را به جلوی لیست اضافه می‌کند. ابتدا، اطمینان حاصل می‌کند که ناحیه داده شده دارای اندازه و تراز لازم برای ذخیره یک `ListNode` است. سپس گره را ایجاد کرده و طی مراحل زیر آن را در لیست قرار می دهد:

![](linked-list-allocator-push.svg)

مرحله 0 وضعیت هیپ را قبل از فراخوانی `add_free_region` نشان می‌دهد. در مرحله 1، متد با ناحیه حافظه که در تصویر به عنوان `freed` مشخص شده است فراخوانی می‌شود. پس از بررسی‌های اولیه، متد یک `node` جدید در هیپ خود با اندازه ناحیه آزاد شده ایجاد می‌کند. سپس از متد [`Option::take`] استفاده می‌کند تا اشاره‌گر `next` گره را روی اشاره‌گر `head` فعلی تنظیم کند، در نتیجه اشاره‌گر `head` را به `None` بازنشانی می‌کند.

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

در مرحله 2، متد، `node` تازه ایجاد شده را از طریق متد [`write`] در ابتدای ناحیه حافظه آزاد شده می‌نویسد. سپس اشاره‌گر `head` را به گره جدید متصل می‌دهد. ساختار اشاره‌گر حاصل کمی آشفته به نظر می‌رسد زیرا ناحیه آزاد شده همیشه در ابتدای لیست درج می‌شود، اما اگر اشاره‌گرها را دنبال کنیم می‌بینیم که هر ناحیه آزاد هنوز از اشاره‌گر `head` قابل دسترسی است.

[`write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

#### متد `find_region`

دومین عملیات اساسی در یک لیست پیوندی یافتن یک ورودی و حذف آن از لیست است. این عملیات مرکزی مورد نیاز برای پیاده‌سازی متد `alloc` است. ما این عملیات را به عنوان متد `find_region` به شیوه زیر پیاده‌سازی می‌کنیم:

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

این متد از یک متغیر `current` و یک حلقه [`while let`] برای تکرار بر روی عناصر لیست استفاده می‌کند. در ابتدا، `current` روی گره `head` (ساختگی) تنظیم شده است. در هر تکرار، سپس به فیلد `next` گره فعلی (در بلوک `else`) به‌روزرسانی می‌شود. اگر ناحیه برای تخصیص با اندازه و تراز داده شده مناسب باشد، ناحیه از لیست حذف شده و همراه با آدرس `alloc_start` برگردانده می‌شود.

[`while let` loop]: https://doc.rust-lang.org/reference/expressions/loop-expr.html#predicate-pattern-loops

وقتی اشاره‌گر `current.next` به `None` برسد، حلقه خارج می‌شود. این بدان معنی است که ما کل لیست را پیمایش کردیم اما هیچ ناحیه‌ای را پیدا نکردیم که برای تخصیص مناسب باشد. در آن صورت، `None` را برمی‌گردانیم. برای بررسی این‌که یک ناحیه مناسب است یا خیر، از تابع `alloc_from_region` استفاده می‌شود که اجرای آن به‌زودی نشان داده می‌شود.

بیایید نگاهی دقیق‌تر به نحوه حذف یک ناحیه مناسب از لیست بیندازیم:

![](linked-list-allocator-remove-region.svg)

مرحله 0 وضعیت را قبل از هر تنظیم اشاره‌گر نشان می‌دهد. ناحیه‌های `region` و `current` و اشاره‌گرهای `region.next` و `current.next` در تصویر مشخص شده‌اند. در مرحله 1، هر دو اشاره‌گر `region.next` و `current.next` با استفاده از متد [`Option::take`] به `None` بازنشانی می‌شوند. اشاره‌گرهای اصلی در متغیرهای محلی به نام `next` و `ret` ذخیره می‌شوند.

در مرحله 2، اشاره‌گر `current.next` روی اشاره‌گر محلی `next` تنظیم می‌شود که اشاره‌گر `region.next` اصلی است. اثر این است که `current` اکنون مستقیماً به ناحیه بعد از `region` اشاره می‌کند، بنابراین `region` دیگر عنصری از لیست پیوندی نیست. سپس این تابع اشاره‌گر را به `region` ذخیره شده در متغیر محلی `ret` برمی‌گرداند.

##### تابع `alloc_from_region`

تابع `alloc_from_region` نشان می‌دهد که آیا یک ناحیه برای تخصیص با اندازه و تراز مشخص مناسب است یا خیر. این‌گونه تعریف می‌شود:

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

ابتدا، تابع آدرس شروع و پایان یک تخصیص بالقوه را با استفاده از تابع `align_up` که قبلا تعریف کردیم و متد [`checked_add`] محاسبه می‌کند. اگر سرریز اتفاق بیفتد یا اگر آدرس انتهایی عقب‌تر از آدرس انتهای ناحیه باشد، تخصیص در ناحیه جا نمی‌شود و یک خطا برمی‌گردانیم.

این تابع پس از آن بررسی‌ای که کمتر واضح است را انجام می‌دهد. این بررسی ضروری است زیرا در اکثر مواقع یک تخصیص با یک ناحیه مناسب مطابقت کامل ندارد، به طوری که بخشی از ناحیه پس از تخصیص قابل استفاده باقی می‌ماند. این قسمت از ناحیه باید `ListNode` خود را پس از تخصیص ذخیره کند، بنابراین باید برای انجام این کار به اندازه کافی بزرگ باشد. بررسی دقیقاً آن را تأیید می‌کند: یا تخصیص کاملاً متناسب است (`excess_size == 0`) یا اندازه اضافی به اندازه‌ای بزرگ است که یک `ListNode` را ذخیره کند.

#### پیاده‌سازی `GlobalAlloc`

با عملیات اساسی ارائه شده توسط متدهای `add_free_region` و `find_region`، اکنون می‌توانیم صفت `GlobalAlloc` را پیاده‌سازی کنیم. مانند تخصیص‌دهنده بامپ، ما این صفت را مستقیماً برای `LinkedListAllocator` پیاده‌سازی نمی‌کنیم، بلکه فقط برای یک `Locked<LinkedListAllocator>` بسته‌بندی شده است. [بسته‌بندی `Locked`] تغییرپذیری داخلی را از طریق یک قفل چرخشی اضافه می‌کند، که به ما امکان می‌دهد نمونه تخصیص‌دهنده را تغییر دهیم حتی اگر متدهای `alloc` و `dealloc` فقط ارجاعات `&self` را دریافت کنند.

[بسته‌بندی `Locked`]: @/edition-2/posts/11-allocator-designs/index.md#a-locked-wrapper-type

پیاده‌سازی به شکل زیر است:

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

بیایید با روش `dealloc` شروع کنیم زیرا ساده‌تر است: ابتدا، برخی از تنظیمات چیدمان را انجام می‌دهد که به‌زودی توضیح خواهیم داد، و با فراخوانی تابع [`Mutex::lock`] روی [بسته‌بندی `Locked`] مرجع `&mut LinkedListAllocator` را بازیابی می‌کند. سپس تابع `add_free_region` را فراخوانی می‌کند تا ناحیه اختصاص داده شده را به لیست آزاد اضافه کند.

متد `alloc` کمی پیچیده‌تر است. با همان تنظیمات چیدمان شروع می‌شود و همچنین تابع [`Mutex::lock`] را برای دریافت یک مرجع تخصیص‌دهنده تغییرپذیر فراخوانی می‌کند. سپس از متد `find_region` برای یافتن یک ناحیه حافظه مناسب برای تخصیص و حذف آن از لیست استفاده می‌کند. اگر این کار موفق نشد و `None` برگردانده شد، `null_mut` را برمی‌گرداند تا خطایی را نشان دهد زیرا ناحیه حافظه مناسبی وجود ندارد.

در صورت موفقیت، متد `find_region` چندین ناحیه مناسب (که دیگر در لیست نیست) و آدرس شروع تخصیص را برمی‌گرداند. با استفاده از `alloc_start`، اندازه تخصیص و آدرس پایان ناحیه، آدرس پایان تخصیص و اندازه اضافی را دوباره محاسبه می‌کند. اگر اندازه اضافی تهی نباشد، `add_free_region` را فراخوانی می‌کند تا اندازه اضافی ناحیه حافظه را دوباره به لیست آزاد اضافه کند. در نهایت، آدرس `alloc_start` را که به عنوان اشاره‌گر `*mut u8` فرستاده شده است، برمی‌گرداند.

#### تنظیمات چیدمان

بنابراین این تنظیمات چیدمان که در ابتدای `alloc` و `dealloc` انجام می‌دهیم چیست؟ آن‌ها اطمینان حاصل می‌کنند که هر بلوک تخصیص یافته قادر به ذخیره `ListNode` است. این مهم است زیرا بلوک حافظه قرار است در نقطه‌ای بازستانی تخصیص (deallocate) شود، جایی که می‌خواهیم یک `ListNode` روی آن بنویسیم. اگر بلوک کوچک‌تر از `ListNode` باشد یا تراز درستی نداشته باشد، رفتار نامشخصی ممکن است رخ دهد.

تنظیمات چیدمان توسط یک تابع `size_align` انجام می‌شود که به صورت زیر تعریف می‌شود:

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

ابتدا، این تابع از متد [`align_to`] در [`Layout`] استفاده می‌کند تا در صورت لزوم، تراز را به تراز یک `ListNode` افزایش دهد. سپس از متد [`pad_to_align`] برای گرد کردن اندازه به مضربی از تراز استفاده می‌کند تا اطمینان حاصل شود که آدرس شروع بلوک حافظه بعدی هم تراز درستی برای ذخیره یک `ListNode` خواهد داشت.
در مرحله دوم از متد [`max`] برای اعمال حداقل اندازه تخصیص `mem::size_of::<ListNode>` استفاده می‌کند. به این ترتیب، تابع `dealloc` می‌تواند با خیال راحت یک `ListNode` در بلوک حافظه آزاد شده بنویسد.

[`align_to`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align_to
[`pad_to_align`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.pad_to_align
[`max`]: https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.max

### استفاده از آن

اکنون می‌توانیم استاتیک `ALLOCATOR` را در ماژول `allocator` به‌روزرسانی کنیم تا از `LinkedListAllocator` جدید خود استفاده کنیم:

```rust
// in src/allocator.rs

use linked_list::LinkedListAllocator;

#[global_allocator]
static ALLOCATOR: Locked<LinkedListAllocator> =
    Locked::new(LinkedListAllocator::new());
```

از آن‌جایی که تابع `init` برای تخصیص‌دهنده‌های بامپ و لیست پیوندی یکسان عمل می‌کند، نیازی به تغییر فراخوان `init` در `init_heap` نداریم.

وقتی اکنون تست‌های `heap_allocation` خود را اجرا می‌کنیم، می‌بینیم که همه تست‌ها، از جمله تست `many_boxes_long_lived` که با تخصیص‌دهنده بامپ ناموفق بود، اکنون با موفقیت انجام می‌شود:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

این نشان می‌دهد که تخصیص‌دهنده لیست پیوندی ما می‌تواند از حافظه آزاد شده برای تخصیص‌های بعدی مجددا استفاده کند.

### گفتگو

بر خلاف تخصیص‌دهنده بامپ، تخصیص‌دهنده لیست پیوندی به عنوان یک تخصیص‌دهنده هدف عمومی بسیار مناسب‌تر است، عمدتاً به این دلیل که قادر به استفاده مجدد مستقیم از حافظه آزاد شده است. با این حال، معایبی نیز دارد. برخی از آن‌ها فقط ناشی از پیاده‌سازی پایه ما هستند، اما اشکالات اساسی در طراحی تخصیص‌دهنده نیز وجود دارد.

#### Merging Freed Blocks

مشکل اصلی پیاده‌سازی ما این است که فقط هیپ را به بلوک‌های کوچکتر تقسیم می‌کند، اما هرگز آن‌ها را دوباره با هم ادغام نمی‌کند. این مثال را در نظر بگیرید:

![](linked-list-allocator-fragmentation-on-dealloc.svg)

در خط اول، سه تخصیص روی هیپ ایجاد می‌شود. دو تا از آن‌ها دوباره در خط 2 و سومی در خط 3 آزاد می‌شوند. اکنون کل هیپ دوباره استفاده نشده است، اما همچنان به چهار بلوک جداگانه تقسیم می‌شود. در این مرحله، یک تخصیص بزرگ ممکن است دیگر امکان‌پذیر نباشد زیرا هیچ یک از چهار بلوک به اندازه کافی بزرگ نیست. با گذشت زمان، این روند ادامه می‌یابد و هیپ به بلوک‌های کوچک‌تر و کوچک‌تر تقسیم می‌شود. در برخی مواقع، پشته آن‌قدر تکه‌تکه می‌شود که حتی یک تخصیص با اندازه معمولی نیز با شکست مواجه می‌شود.

برای رفع این مشکل، باید بلوک‌های آزاد شده مجاور را دوباره با هم ادغام کنیم. برای مثال بالا، این به معنی حالت زیر است:

![](linked-list-allocator-merge-on-dealloc.svg)

مانند قبل، دو مورد از سه تخصیص در ردیف `2`  آزاد می‌شود. به‌جای نگه داشتن هیپ تکه‌تکه شده، اکنون یک مرحله اضافی در خط `2a` انجام می‌دهیم تا دو بلوک سمت راست را دوباره با هم ادغام کنیم. در خط `3`، تخصیص سوم آزاد می‌شود (مانند قبل)، و در نتیجه یک هیپ کاملاً بدون استفاده به‌دست می‌آید که با سه بلوک مجزا نشان داده می‌شود. در یک مرحله ادغام اضافی در خط `3a`، سپس سه بلوک مجاور را دوباره با هم ادغام می‌کنیم.

کریت `linked_list_allocator` این استراتژی ادغام را به روش زیر پیاده‌سازی می‌کند: به جای قرار دادن بلوک‌های حافظه آزاد شده در ابتدای لیست پیوندی در `deallocate`، همیشه لیست را مرتب‌شده بر اساس آدرس شروع نگه می‌دارد. به این ترتیب، ادغام می‌تواند مستقیماً در فراخوانی `deallocate` با بررسی آدرس‌ها و اندازه‌های دو بلوک همسایه در لیست انجام شود. البته عملیات بازستانی تخصیص از این طریق کندتر است، اما از تکه‌تکه شدن هیپ همان‌طور که در بالا دیدیم جلوگیری می‌کند.

#### کارایی

همان‌طور که در بالا آموختیم، تخصیص‌دهنده بامپ بسیار سریع است و می‌تواند به شکلی بهینه شود که تنها چند عملیات اسمبلی باشد. تخصیص‌دهنده لیست پیوندی در این زمینه کارایی بسیار بدتری دارد. مشکل این است که یک درخواست تخصیص ممکن است نیاز داشته باشد که کل لیست پیوند شده را تا زمانی که یک بلوک مناسب پیدا کند، طی کند.

از آن‌جایی که طول لیست به تعداد بلوک‌های حافظه استفاده نشده بستگی دارد، کارایی می‌تواند برای برنامه‌های مختلف بسیار متفاوت باشد. برنامه‌ای که فقط چند تخصیص ایجاد می‌کند، کارایی تخصیص نسبتاً سریعی را تجربه خواهد کرد. با این حال، برای برنامه‌ای که هیپ را با تخصیص‌های زیاد تکه‌تکه می‌کند، کارایی تخصیص بسیار بد خواهد بود زیرا لیست پیوند شده بسیار طولانی است و عمدتاً شامل بلوک‌های بسیار کوچک است.

شایان ذکر است که این مشکل کارایی یک مشکل ناشی از پیاده‌سازی اولیه ما نیست، بلکه یک مشکل اساسی از رویکرد لیست پیوندی است. از آن‌جایی که کارایی تخصیص می‌تواند برای کدهای سطح هسته بسیار مهم باشد، در ادامه طرح سومی را بررسی می‌کنیم که کارایی را بهبود داده اما باعث کاهش استفاده از حافظه می‌شود.

## تخصیص‌دهنده با اندازه بلوک ثابت

در ادامه، یک طراحی تخصیص دهنده ارائه می‌کنیم که از بلوک‌های حافظه با اندازه ثابت برای انجام درخواست‌های تخصیص استفاده می‌کند. به این ترتیب، تخصیص‌دهنده اغلب بلوک‌هایی را برمی‌گرداند که بزرگ‌تر از مقدار مورد نیاز برای تخصیص‌ها هستند، که منجر به هدر رفتن حافظه به دلیل [تکه‌تکه شدن داخلی] می‌شود. از طرف دیگر، زمان مورد نیاز برای یافتن یک بلوک مناسب (در مقایسه با تخصیص‌دهنده لیست پیوندی) را به شدت کاهش می‌دهد و در نتیجه کارایی تخصیص بسیار بهتری دارد.

### مقدمه

ایده پشت یک _تخصیص‌دهنده با اندازه بلوک ثابت_ به شرح زیر است: به جای تخصیص دقیقاً همان مقدار حافظه که درخواست شده است، تعداد کمی از اندازه‌های بلوک را تعریف می‌کنیم و هر تخصیص را به اندازه بلوک بعدی جمع می‌کنیم. به عنوان مثال، با اندازه بلوک‌های 16، 64 و 512 بایت، یک تخصیص 4 بایتی یک بلوک 16 بایتی را برمی‌گرداند، همین‌طور یک تخصیص 48 بایتی یک بلوک 64 بایتی را و یک تخصیص 128 بایتی یک بلوک 512 بایتی را برمی‌گرداند. .

مانند تخصیص‌دهنده لیست پیوندی، ما با ایجاد یک لیست پیوندی در حافظه استفاده نشده، حافظه استفاده نشده را پیگیری می‌کنیم. با این حال، به جای استفاده از یک لیست واحد با اندازه بلوک‌های مختلف، یک لیست جداگانه برای هر کلاس اندازه ایجاد می‌کنیم. سپس هر لیست فقط بلوک‌های اندازه مربوط به خودش را ذخیره می‌کند. به عنوان مثال، با اندازه‌های بلوک 16، 64، و 512، سه لیست پیوندی جداگانه در حافظه وجود خواهد داشت:

![](fixed-size-block-example.svg).

به جای یک اشاره‌گر `head`، سه اشاره‌گر هِد `head_64`، `head_16` و `head_512` داریم که هر کدام به اولین بلوک استفاده نشده با اندازه مربوطه اشاره می‌کنند. همه گره‌ها در یک لیست واحد اندازه یکسانی دارند. به عنوان مثال، فهرستی که با اشاره‌گر `head_16` شروع می‌شود، فقط شامل بلوک‌های 16 بایتی است. یعنی ما دیگر نیازی به ذخیره اندازه در هر گره لیست نداریم زیرا قبلاً با نام اشاره‌گر هد مشخص شده است.

از آن‌جایی که هر عنصر در یک لیست اندازه یکسانی دارد، هر عنصر لیست به همان اندازه برای درخواست تخصیص مناسب است. یعنی ما می‌توانیم با استفاده از مراحل زیر یک تخصیص را بسیار کارآمد انجام دهیم:

- اندازه تخصیص درخواستی را به اندازه بلوک بعدی گرد کنید. به عنوان مثال، زمانی که تخصیص 12 بایتی درخواست می‌شود، اندازه بلوک 16 بایتی را انتخاب می‌کنیم.
- اشاره‌گر هد لیست را بازیابی کنید، به عنوان مثال، از یک آرایه. برای اندازه بلوک 16، باید از `head_16` استفاده کنیم.
- اولین بلوک را از لیست حذف کرده و برگردانید.

مهم‌تر از همه، ما همیشه می‌توانیم اولین عنصر لیست را برگردانیم و دیگر نیازی به طی کردن کل لیست نداریم. بنابراین، تخصیص‌ها بسیار سریعتر از تخصیص‌دهنده لیست پیوندی است.

#### اندازه بلوک و حافظه هدر رفته

بسته به اندازه بلوک‌ها، با گرد کردن به سمت بالا، حافظه زیادی را از دست می‌دهیم. به عنوان مثال، هنگامی که یک بلوک 512 بایتی برای یک تخصیص 128 بایتی برگردانده می‌شود، سه چهارم حافظه تخصیص یافته استفاده نمی‌شود. با تعریف اندازه بلوک‌های معقول، می‌توان مقدار حافظه تلف شده را تا حدی محدود کرد. به عنوان مثال، هنگام استفاده از توان‌های 2 (4، 8، 16، 32، 64، 128، ...) به عنوان اندازه بلوک، می‌توانیم اتلاف حافظه را در بدترین حالت به نصف اندازه تخصیص و در حالت متوسط به یک چهارم تخصیص محدود کنیم.

همچنین بهینه‌سازی اندازه بلوک‌ها بر اساس اندازه‌های تخصیص رایج در یک برنامه، رایج است. به عنوان مثال، می‌توانیم اندازه بلوک 24 را برای بهبود استفاده از حافظه برای برنامه‌هایی که اغلب تخصیص‌های 24 بایتی را انجام می‌دهند، اضافه کنیم. به این ترتیب، مقدار حافظه تلف شده را اغلب می‌توان بدون از دست دادن مزایای کارایی کاهش داد.

#### بازستانی تخصیص

مانند تخصیص دادن، بازستانی تخصیص نیز بسیار کارآمد است. این شامل مراحل زیر است:

- اندازه تخصیص آزاد شده را به اندازه بلوک بعدی گرد کنید. این مورد ضروری است زیرا کامپایلر فقط اندازه تخصیص درخواستی را به `dealloc` منتقل می‌کند، نه اندازه بلوکی که توسط `alloc` برگردانده شده است. با استفاده از یک تابع تنظیم اندازه در `alloc` و `dealloc` می‌توانیم مطمئن شویم که همیشه مقدار صحیح حافظه را آزاد می‌کنیم.
- اشاره‌گر هدِ لیست را بازیابی کنید، به عنوان مثال، از یک آرایه.
- بلوک آزاد شده را با به‌روزرسانی اشاره‌گر هد به جلوی لیست اضافه کنید.

مهمتر از همه، هیچ پیمایشی از لیست برای بازستانی تخصیص لازم نیست. این بدان معناست که زمان مورد نیاز برای فراخوانی `dealloc` بدون در نظر گرفتن طول لیست ثابت می‌ماند.

#### تخصیص‌دهنده جایگزین (Fallback)

با توجه به این‌که تخصیص‌های بزرگ (بزرگ‌تر از 2KB) اغلب نادر هستند، به‌ویژه در هسته‌های سیستم‌عامل، ممکن است منطقی باشد که از تخصیص‌دهنده‌ای متفاوت برای این تخصیص‌ها استفاده کنیم. به عنوان مثال، می‌توانیم برای کاهش اتلاف حافظه، از یک تخصیص‌دهنده لیست پیوندی برای تخصیص‌های بیشتر از 2048 بایت استفاده کنیم. از آن‌جایی که تنها تعداد بسیار کمی تخصیص با آن اندازه مورد انتظار است، لیست پیوندی کوچک باقی می‌ماند تا بازستانی تخصیص همچنان به طور معقولی سریع باشد.

#### ایجاد بلوک‌های جدید

در بالا، ما همیشه فرض می‌کردیم که همیشه بلوک‌های کافی با یک اندازه خاص در لیست وجود دارد تا تمام درخواست‌های تخصیص را برآورده کند. با این حال، در برخی موارد لیست پیوندی برای یک اندازه بلوک خاص خالی می‌شود. در این مرحله، دو راه وجود دارد که چگونه می‌توانیم بلوک‌های بلااستفاده جدید با اندازه‌ای خاص برای انجام یک درخواست تخصیص ایجاد کنیم:

- یک بلوک جدید از تخصیص‌دهنده جایگزین (در صورت وجود) اختصاص دهید.
- یک بلوک بزرگ‌تر را از یک لیست که مربوط به اندازه‌های بزرگ‌تر هست را تقسیم کنید. اگر اندازه بلوک توان دو باشد، این بهترین گزینه خواهد بود. به عنوان مثال، یک بلوک 32 بایتی را می‌توان به دو بلوک 16 بایتی تقسیم کرد.

برای پیاده‌سازی خود، بلوک‌های جدیدی را از تخصیص‌دهنده جایگزین تخصیص می‌دهیم زیرا پیاده‌سازی بسیار ساده‌تر است.

### پیاده‌سازی

اکنون که می‌دانیم یک تخصیص‌دهنده با اندازه بلوک ثابت چگونه کار می‌کند، می‌توانیم پیاده‌سازی خود را شروع کنیم. ما به پیاده‌سازی تخصیص‌دهنده لیست پیوندی ایجاد شده در بخش قبل وابسته نیستیم، بنابراین می‌توانید این قسمت را دنبال کنید حتی اگر پیاده‌سازی تخصیص‌دهنده لیست پیوندی را نادیده گرفته باشید.

#### لیست گره

ما پیاده‌سازی خود را با ایجاد یک نوع `ListNode` در یک ماژول جدید `allocator::fixed_size_block` شروع می‌کنیم:

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

این نوع شبیه به نوع `ListNode` در [پیاده‌سازی تخصیص‌دهنده لیست پیوندی] ما است، با این تفاوت که فیلد دوم `size` نداریم. فیلد `size` مورد نیاز نیست زیرا هر بلوک در یک لیست با طراحی تخصیص‌دهنده با اندازه بلوک ثابت اندازه یکسانی دارد.

[پیاده‌سازی تخصیص‌دهنده لیست پیوندی]: #the-allocator-type

#### اندازه‌های بلوک

در مرحله بعد، یک برش `BLOCK_SIZES` ثابت با اندازه‌های بلوک مورد استفاده برای پیاده‌سازی تعریف می‌کنیم:

```rust
// in src/allocator/fixed_size_block.rs

/// The block sizes to use.
///
/// The sizes must each be power of 2 because they are also used as
/// the block alignment (alignments must be always powers of 2).
const BLOCK_SIZES: &[usize] = &[8, 16, 32, 64, 128, 256, 512, 1024, 2048];
```

به عنوان اندازه بلوک، از توان‌های 2 استفاده می‌کنیم که از 8 شروع و تا 2048  ادامه دارد. ما هیچ اندازه بلوکی کوچک‌تر از 8 تعریف نمی‌کنیم زیرا هر بلوک باید قابلیت ذخیره یک اشاره‌گر 64 بیتی را در بلوک بعدی داشته باشد. برای تخصیص‌های بیشتر از 2048 بایت، ما از یک تخصیص‌دهنده لیست پیوندی استفاده می‌کنیم.

برای ساده‌سازی پیاده‌سازی، تعریف می‌کنیم که اندازه یک بلوک برابر با تراز مورد نیاز آن در حافظه است. بنابراین یک بلوک 16 بایتی همیشه روی یک مرز 16 بایتی و یک بلوک 512 بایتی روی یک مرز 512 بایتی تراز می‌شود. از آن‌جایی که ترازها همیشه باید توان 2 باشند، این امر هر اندازه بلوک دیگری را رد می‌کند. اگر در آینده به اندازه بلوک‌هایی نیاز داشته باشیم که توان 2 نباشد، همچنان می‌توانیم پیاده‌سازی خود را برای این کار تنظیم کنیم (مثلاً با تعریف یک آرایه `BLOCK_ALIGNMENTS` دوم).

#### نوع تخصیص‌دهنده

با استفاده از نوع `ListNode` و برش `BLOCK_SIZES`، اکنون می‌توانیم نوع تخصیص‌دهنده خود را تعریف کنیم:

```rust
// in src/allocator/fixed_size_block.rs

pub struct FixedSizeBlockAllocator {
    list_heads: [Option<&'static mut ListNode>; BLOCK_SIZES.len()],
    fallback_allocator: linked_list_allocator::Heap,
}
```

فیلد `list_heads` آرایه‌ای از اشاره‌گرهای `head` است، یکی برای هر اندازه بلوک. این با استفاده از `len()` از برش `BLOCK_SIZES` به عنوان طول آرایه پیاده‌سازی می‌شود. به‌عنوان یک تخصیص‌دهنده جایگزین برای تخصیص‌های بزرگ‌تر از بزرگ‌ترین اندازه بلوک، از تخصیص‌دهنده ارائه‌شده توسط `linked_list_allocator` استفاده می‌کنیم. همچنین می‌توانیم از `LinkedListAllocator` که خودمان پیاده‌سازی کرده‌ایم به جای آن استفاده کنیم، اما این عیب را دارد که [بلوک‌های آزادشده] را ادغام نمی‌کند.

[merge freed blocks]: #merging-freed-blocks

برای ساخت یک `FixedSizeBlockAllocator`، همان توابع `new` و `init` را ارائه می‌کنیم که برای سایر انواع تخصیص‌دهنده نیز پیاده‌سازی کردیم:

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

تابع `new` فقط آرایه `list_heads` را با گره‌های خالی مقداردهی می‌کند و یک تخصیص‌دهنده لیست پیوندی [`empty`] را به‌عنوان `fallback_allocator` ایجاد می‌کند. ثابت `EMPTY` مورد نیاز است زیرا به کامپایلر راست می‌گوییم که می‌خواهیم آرایه را با مقدار ثابت مقداردهی اولیه کنیم. مقداردهی اولیه آرایه مستقیماً به صورت `[None; BLOCK_SIZES.len()]` کار نمی‌کند، زیرا کامپایلر نیاز دارد که `Option<&'static mut ListNode>` صفت `Copy` را پیاده‌سازی کند، که این کار را نمی‌کند. این محدودیت فعلی کامپایلر راست است که ممکن است در آینده از بین برود.

[`empty`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.empty

اگر قبلاً این کار را برای پیاده‌سازی `LinkedListAllocator` انجام نداده‌اید، باید **`#![feature(const_mut_refs)]`** را به ابتدای `lib.rs` خود اضافه کنید. زیرا هرگونه استفاده از انواع مرجع تغییرپذیر در توابع const همچنان ناپایدار است، از جمله نوع عنصر آرایه `Option<&'static mut ListNode>` در فیلد `list_heads` (حتی اگر آن را روی `None` تنظیم کنیم).

تابع ناامن `init` فقط تابع [`init`] از `fallback_allocator` را بدون انجام مقداردهی اولیه اضافیِ آرایه `list_heads` فراخوانی می‌کند. درعوض، ما لیست‌ها را به صورت تنبلانه (lazily) در فراخوانی‌های `alloc` و `dealloc` مقداردهی اولیه می‌کنیم.

[`init`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.init

برای راحتی، ما همچنین یک متد `fallback_alloc` خصوصی ایجاد می‌کنیم که با استفاده از `fallback_allocator` عملیات تخصیص را انجام می‌دهد:

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

از آن‌جایی که نوع [`Heap`] از کریت `linked_list_allocator`، صفت [`GlobalAlloc`] را پیاده‌سازی نمی‌کند (زیرا [بدون عملیات قفل کردن ممکن نیست]). در عوض، یک متد [`allocate_first_fit`] ارائه می‌کند که رابط کاربری کمی متفاوت دارد. به جای برگرداندن `*mut u8` و استفاده از اشاره‌گر تهی برای سیگنال دادن یک خطا، `Result<NonNull<u8>، ()>` را برمی‌گرداند. نوع [`NonNull`] یک انتزاع برای یک اشاره‌گر خام است که تضمین شده است اشاره‌گر تهی نباشد. با نگاشت حالت `Ok` به متد [`NonNull::as_ptr`] و حالت `Err` به یک اشاره‌گر تهی، می‌توانیم به راحتی آن را به نوع `*mut u8` برگردانیم.

[`Heap`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html
[not possible without locking]: #globalalloc-and-mutability
[`allocate_first_fit`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.allocate_first_fit
[`NonNull`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html
[`NonNull::as_ptr`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html#method.as_ptr

#### محاسبه کردن اندیس لیست

قبل از اینکه صفت `GlobalAlloc` را پیاده‌سازی کنیم، یک تابع کمکی `list_index` تعریف می‌کنیم که کم‌ترین اندازه بلوک ممکن را برای یک [`Layout`] داده شده را برمی‌گرداند:

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

بلوک باید حداقل اندازه و تراز مورد نیاز `Layout` را داشته باشد. از آن‌جایی که ما تعریف کردیم که اندازه هر بلوک برابر با تراز آن است، به این معنی است که `required_block_size` برابر با [بیش‌ترین مقدار] ویژگی‌های [`size()`] و [`align()`] طرح است. برای یافتن بلوک بزرگ‌تر بعدی در برش `BLOCK_SIZES`، ابتدا از متد [`iter()`] برای بدست آوردن یک iterator و سپس از متد [`position()`] برای یافتن اندیس اولین بلوک استفاده می‌کنیم که حداقل به اندازه `required_block_size` است.

[بیش‌ترین مقدار]: https://doc.rust-lang.org/core/cmp/trait.Ord.html#method.max
[`size()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.size
[`align()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align
[`iter()`]: https://doc.rust-lang.org/std/primitive.slice.html#method.iter
[`position()`]:  https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.position

توجه داشته باشید که ما خودِ اندازه بلوک را برنمی‌گردانیم، بلکه اندیس `BLOCK_SIZES` برمی‌گردانیم. زیرا می‌خواهیم از اندیسی که برگشت داده شده را به عنوان اندیسی در آرایه `list_heads` استفاده کنیم.

#### پیاده‌سازی `GlobalAlloc`

آخرین مرحله، پیاده‌سازی صفت `GlobalAlloc` است:

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

مانند سایر تخصیص‌دهنده‌ها، صفت `GlobalAlloc` را مستقیماً برای نوع تخصیص‌دهنده خود پیاده‌سازی نمی‌کنیم، اما از [بسته‌بندی `Locked`] برای افزودن قابلیت تغییرپذیری داخلی همگام‌سازی شده استفاده می‌کنیم. از آن‌جایی که پیاده‌سازی‌های `alloc` و `dealloc` نسبتاً بزرگ هستند، در ادامه آن‌ها را یکی‌یکی معرفی می‌کنیم.

##### `alloc`

پیاده‌سازی متد alloc به شکل زیر است:

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

بیایید مرحله به مرحله آن را مرور کنیم:

ابتدا، از متد `Locked::lock` برای دریافت یک مرجع تغییرپذیر به نمونه تخصیص‌دهنده بسته‌بندی شده استفاده می‌کنیم. سپس، تابع `list_index` را که به تازگی تعریف کرده‌ایم فراخوانی می‌کنیم تا اندازه بلوک مناسب برای چیدمان داده شده را محاسبه کنیم و اندیس مربوطه را در آرایه `list_heads` قرار دهیم. اگر این اندیس `None` باشد، هیچ اندازه بلوکی برای تخصیص مناسب نیست، بنابراین ما از `fallback_allocator` با استفاده از تابع `fallback_alloc` استفاده می‌کنیم.

اگر اندیس لیست `Some` باشد، سعی می‌کنیم اولین گره را در لیست مربوطه که با `list_heads[index]` شروع شده است، با استفاده از متد [`Option::take] حذف کنیم. اگر لیست خالی نباشد، وارد شاخه `Some(node)` عبارت `match` می‌شویم، جایی که اشاره‌گر هد لیست را به سمت جانشین `node` پاپ شده نشان می‌دهیم (با استفاده از مجدد [`take`][`Option::take`]). در نهایت، اشاره‌گر `node` پاپ شده را به‌عنوان `*mut u8` برمی‌گردانیم.

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

اگر هد لیست `None` باشد، نشان‌دهنده خالی بودن لیست بلوک‌ها است. یعنی ما باید یک بلوک جدید را همان‌طور که [در بالا توضیح داده شد](#creating-new-blocks) بسازیم. برای آن، ابتدا اندازه بلوک فعلی را از برش `BLOCK_SIZES` دریافت می‌کنیم و از آن به عنوان اندازه و تراز بلوک جدید استفاده می‌کنیم. سپس یک `Layout` جدید از آن ایجاد می‌کنیم و متد `fallback_alloc` را برای انجام تخصیص فراخوانی می‌کنیم. دلیل تنظیم چیدمان و تراز این است که بلوک در زمان بازستانی تخصیص به لیست بلوک اضافه می‌شود.

#### `dealloc`

پیاده‌سازی متد `dealloc` به شکل زیر است:

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

مانند `alloc`، ابتدا از متد `lock` برای به دست آوردن یک مرجع تخصیص‌دهنده تغییرپذیر و سپس از تابع `list_index` برای دریافت لیست بلوک مربوط به `Layout` استفاده می‌کنیم. اگر اندیس `None` باشد، هیچ اندازه بلوک مناسبی در `BLOCK_SIZES` وجود ندارد، که نشان می‌دهد تخصیص توسط تخصیص‌دهنده جایگزین ایجاد شده است. بنابراین ما از [`deallocate`][`Heap::deallocate`] آن برای آزاد کردن مجدد حافظه استفاده می‌کنیم. متد، به‌جای `*mut u8` انتظار یک [`NonNull`] را دارد، بنابراین ابتدا باید اشاره‌گر را تبدیل کنیم. (فراخوانی `unwrap` تنها زمانی با شکست مواجه می‌شود که اشاره‌گر null باشد، که هرگز نباید زمانی که کامپایلر `dealloc` را فراخوانی می‌کند، اتفاق بیفتد.)

[`Heap::deallocate`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.deallocate

اگر `list_index` یک اندیس بلوک را برمی‌گرداند، باید بلوک حافظه آزاد شده را به لیست اضافه کنیم. برای این کار، ابتدا یک `ListNode` جدید ایجاد می‌کنیم که به هد لیست فعلی اشاره می‌کند (با استفاده مجدد از [`Option::take`]). قبل از این‌که گره جدید را در بلوک حافظه آزاد شده بنویسیم، ابتدا اثبات (assert) می‌کنیم که اندازه بلوک فعلی مشخص شده توسط `index` دارای اندازه و تراز مورد نیاز برای ذخیره یک `ListNode` است. سپس عملیات نوشتن را با تبدیل اشاره‌گر `*mut u8` به اشاره‌گر `*mut ListNode` و سپس فراخوانی متد ناامن [`write`][`pointer::write`] روی آن انجام می‌دهیم. آخرین مرحله این است که اشاره‌گر هد لیست را که در حال حاضر `None` است، از آن‌جایی که `take` را روی آن صدا زده‌ایم، روی `ListNode` که به تازگی نوشته شده‌ایم، تنظیم کنیم. برای این کار، `new_node_ptr` خام را به یک مرجع تغییرپذیر تبدیل می‌کنیم.

[`pointer::write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

چند نکته قابل توجه است:

- ما بین بلوک‌های تخصیص‌یافته از یک لیست بلوک و بلوک‌های اختصاص داده شده از تخصیص‌دهنده جایگزین تفاوتی قائل نمی‌شویم. یعنی بلوک‌های جدید ایجاد شده در `alloc` به لیست بلوک در `dealloc` اضافه می‌شوند و در نتیجه تعداد بلوک‌های آن اندازه افزایش می‌یابد.
- متد `alloc` تنها جایی است که بلوک‌های جدید در پیاده‌سازی ما ایجاد می‌شود. یعنی ما ابتدا با لیست بلوک‌های خالی شروع می‌کنیم و تنها زمانی که تخصیص برای آن اندازه بلوک انجام می‌شود، لیست‌ها را با تنبلی پر می‌کنیم.
- ما به بلوک‌های `unsafe` در `alloc` و `dealloc` نیازی نداریم، حتی اگر برخی از عملیات `unsafe` را انجام دهیم. زیرا راست در حال حاضر با کل تابع‌های ناامن به عنوان یک بلوک بزرگ `unsafe` رفتار می‌کند. از آن‌جایی که استفاده از بلوک‌های `unsafe` صریح این مزیت را دارد که مشخص است کدام عملیات ناامن است و کدام نه، یک [proposed RFC](https://github.com/rust-lang/rfcs/pull/2585) برای تغییر این رفتار وجود دارد.

### استفاده کردن از آن

برای استفاده از `FixedSizeBlockAllocator` جدید، باید استاتیک `ALLOCATOR` را در ماژول `allocator` به‌روزرسانی کنیم:

```rust
// in src/allocator.rs

use fixed_size_block::FixedSizeBlockAllocator;

#[global_allocator]
static ALLOCATOR: Locked<FixedSizeBlockAllocator> = Locked::new(
    FixedSizeBlockAllocator::new());
```

از آن‌جایی که تابع `init` برای همه تخصیص‌دهنده‌هایی که پیاده‌سازی کردیم یکسان عمل می‌کند، نیازی نیست فراخوانی `init` را در `init_heap` تغییر دهیم.

وقتی دوباره تست‌های `heap_allocation` خود را اجرا می‌کنیم، همه تست‌ها همچنان باید موفقیت‌آمیز باشند:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

به نظر می رسد تخصیص‌دهنده جدید ما کار می‌کند!

### گفتگو

در حالی که رویکرد بلوک با اندازه ثابت کارایی بسیار بهتری نسبت به رویکرد لیست پیوندی دارد، در هنگام استفاده از توان‌های 2 به عنوان اندازه بلوک، تا نیمی از حافظه را هدر می‌دهد. این‌که آیا این مبادله ارزش آن را دارد یا نه، به شدت به نوع برنامه بستگی دارد. برای هسته سیستم‌عامل، که در آن کارایی بسیار مهم است، به نظر می‌رسد رویکرد بلوک با اندازه ثابت انتخاب بهتری باشد.

در سمت پیاده‌سازی، چیزهای مختلفی وجود دارد که می‌توانیم در پیاده‌سازی فعلی خود بهبود بخشیم:

- به جای تخصیص تنبلانه بلوک‌ها با استفاده از تخصیص‌دهنده جایگزین، شاید بهتر باشد برای بهبود کارایی تخصیص‌های اولیه، لیست‌ها را از قبل پر کنید.
- برای ساده‌تر کردن پیاده‌سازی، ما فقط به اندازه‌های بلوک‌هایی که توان ۲ هستند اجازه دادیم تا بتوانیم از آن‌ها به عنوان تراز بلوک نیز استفاده کنیم. با ذخیره (یا محاسبه) تراز به روشی متفاوت، می‌توانیم اندازه‌های دلخواه دیگری را برای اندازه بلوک مجاز کنیم. به این ترتیب، می‌توانیم اندازه بلوک‌های بیشتری را اضافه کنیم، به عنوان مثال، برای اندازه‌های تخصیص رایج، به منظور به حداقل رساندن حافظه تلف شده.
- ما در حال حاضر فقط بلوک‌های جدید ایجاد می‌کنیم، اما هرگز دوباره آن‌ها را آزاد نمی‌کنیم. این منجر به تکه‌تکه شدن می‌شود و ممکن است در نهایت منجر به شکست تخصیص برای تخصیص‌های بزرگ شود. ممکن است منطقی باشد که برای هر اندازه بلوک یک حداکثر طول لیست اعمال شود. وقتی به حداکثر طول رسید، توزیع‌های بعدی با استفاده از تخصیص‌دهنده جایگزین به‌جای اضافه شدن به لیست آزاد می‌شوند.
- به‌جای بازگشت به یک تخصیص‌دهنده لیست پیوندی، می‌توانیم یک تخصیص‌دهنده ویژه برای تخصیص‌های بیشتر از 4KiB داشته باشیم. ایده این است که از [صفحه‌بندی] که در صفحات 4KiB عمل می‌کند، برای نگاشت یک بلوک پیوسته از حافظه مجازی به فریم‌های فیزیکی غیرپیوسته استفاده شود. به این ترتیب، تکه‌تکه شدن حافظه استفاده نشده دیگر مشکلی برای تخصیص‌های بزرگ نیست.
- با چنین تخصیص‌دهنده صفحه‌ای، ممکن است منطقی باشد که اندازه بلوک را تا 4KiB اضافه کنید و تخصیص‌دهنده لیست پیوند شده را به طور کامل حذف کنید. مزایای اصلی این امر کاهش پراکندگی و بهبود قابلیت پیش‌بینی کارایی، یعنی کارایی بهتر برای بدترین موارد است.

[صفحه‌بندی]: @/edition-2/posts/08-paging-introduction/index.md

توجه به این نکته مهم است که بهبودهای پیاده‌سازی که در بالا ذکر شد فقط پیشنهاد هستند. تخصیص‌دهنده‌های مورد استفاده در هسته‌های سیستم‌عامل معمولاً برای حجم کاری خاص هسته بسیار بهینه‌سازی شده‌اند، که تنها از طریق پروفایل‌سازی گسترده امکان‌پذیر است.

### تغییرات

همچنین تغییرات زیادی در طراحی تخصیص‌دهنده با اندازه بلوک ثابت وجود دارد. دو مثال محبوب عبارتند از _slab allocator_ و _buddy allocator_ که در هسته‌های محبوبی مانند لینوکس نیز استفاده می‌شوند. در ادامه به معرفی کوتاه این دو طرح می‌پردازیم.

#### تخصیص‌دهنده اسلب (Slab)

ایده پشت [تخصیص‌دهنده اسلب] استفاده از اندازه‌های بلوک است که مستقیماً با انواع انتخاب شده در هسته مطابقت دارد. به این ترتیب، تخصیص آن نوع دقیقاً متناسب با اندازه بلوک است و هیچ حافظه‌ای تلف نمی‌شود. گاهی اوقات، حتی ممکن است برای بهبود بیشتر کارایی، نمونه‌های نوع را در بلوک‌های استفاده نشده از قبل مقداردهی کنیم.

[تخصیص‌دهنده اسلب]: https://en.wikipedia.org/wiki/Slab_allocation

تخصیص اسلب اغلب با تخصیص‌دهنده‌های دیگر ترکیب می‌شود. به عنوان مثال، می‌توان آن را همراه با یک تخصیص‌دهنده بلوک با اندازه ثابت برای تقسیم بیشتر یک بلوک اختصاص داده شده به منظور کاهش اتلاف حافظه استفاده کرد. همچنین اغلب برای پیاده‌سازی [object pool pattern] بر روی یک تخصیص بزرگ استفاده می‌شود.

[object pool pattern]: https://en.wikipedia.org/wiki/Object_pool_pattern

#### تخصیص‌دهنده بادی (Buddy)

به جای استفاده از یک لیست پیوندی برای مدیریت بلوک‌های آزاد شده، طراحی [تخصیص‌دهنده بادی] از ساختار داده [درخت باینری] همراه با اندازه بلوک‌های توان ۲ استفاده می‌کند. هنگامی که یک بلوک جدید با یک اندازه خاص مورد نیاز است، یک بلوک با اندازه بزرگ‌تر را به دو نیمه تقسیم می‌کند و در نتیجه دو گره فرزند در درخت ایجاد می‌کند. هر زمان که یک بلوک دوباره آزاد شد، بلوک همسایه در درخت آنالیز می‌شود. اگر همسایه نیز آزاد باشد، دو بلوک دوباره به هم متصل می‌شوند تا بلوکی با اندازه دو برابر ایجاد شود.

مزیت این فرآیند ادغام این است که [تکه‌تکه شدن خارجی] کاهش می‌یابد به طوری که می‌توان از بلوک‌های کوچک آزاد شده برای یک تخصیص بزرگ دوباره استفاده کرد. همچنین از تخصیص‌دهنده جایگزین استفاده نمی‌کند، بنابراین کارایی قابل پیش‌بینی‌تری دارد. بزرگ‌ترین ایراد این است که فقط اندازه بلوک‌های توان 2 امکان‌پذیر است، که ممکن است منجر به مقدار زیادی از حافظه هدر رفته به دلیل [تجزیه داخلی] شود. به همین دلیل، تخصیص‌دهنده‌های دوستان اغلب با یک تخصیص‌دهنده اسلب ترکیب می‌شوند تا یک بلوک اختصاص‌یافته را به چندین بلوک کوچک‌تر تقسیم کنند.

[تخصیص‌دهنده بادی]: https://en.wikipedia.org/wiki/Buddy_memory_allocation
[درخت باینری]: https://en.wikipedia.org/wiki/Binary_tree
[تکه‌تکه شدن خارجی]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation
[تکه‌تکه شدن داخلی]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation


## خلاصه

این پست مروری بر طرح‌های مختلف تخصیص‌دهنده ارائه کرد. ما یاد گرفتیم که چگونه یک [تخصیص‌دهنده بامپ] را پیاده‌سازی کنیم، که با افزایش یک اشاره‌گر `next`، حافظه را به صورت خطی توزیع می‌کند. در حالی که تخصیص بامپ بسیار سریع است، تنها پس از آزاد شدن همه تخصیص‌ها می‌تواند از حافظه مجدداً استفاده کند. به همین دلیل، به ندرت به عنوان یک تخصیص‌دهنده سراسری استفاده می‌شود.

[تخصیص‌دهنده بامپ]: @/edition-2/posts/11-allocator-designs/index.md#bump-allocator

در مرحله بعد، ما یک [تخصیص‌دهنده لیست پیوندی] ایجاد کردیم که از خود بلوک‌های حافظه آزاد شده برای ایجاد یک لیست پیوندی، به اصطلاح [لیست رایگان] استفاده می‌کند. این لیست امکان ذخیره تعداد دلخواه بلوک آزاد شده با اندازه‌های مختلف را فراهم می‌کند. در حالی که هیچ اتلاف حافظه رخ نمی‌دهد، این رویکرد از کارایی ضعیف رنج می‌برد زیرا درخواست تخصیص ممکن است به پیمایش کامل لیست نیاز داشته باشد. پیاده‌سازی ما همچنین از [تکه‌تکه شدن خارجی] رنج می‌برد زیرا بلوک‌های آزاد شده مجاور را دوباره با هم ادغام نمی‌کند.

[تخصیص‌دهنده لیست پیوندی]: @/edition-2/posts/11-allocator-designs/index.md#linked-list-allocator
[لیست رایگان]: https://en.wikipedia.org/wiki/Free_list

برای رفع مشکلات کارایی رویکرد لیست پیوندی، یک [تخصیص‌دهنده با اندازه بلوک ثابت] ایجاد کردیم که مجموعه‌ای ثابت از اندازه بلوک را از پیش تعریف می‌کند. برای هر اندازه بلوک، یک [لیست رایگان] مجزا وجود دارد، به طوری که تخصیص‌ها و بازستانی تخصیص‌ها فقط باید در قسمت جلوی لیست اضافه/پاپ شوند و بنابراین بسیار سریع هستند. از آن‌جایی که هر تخصیص به اندازه بلوک بزرگ‌تر بعدی گرد می‌شود، مقداری از حافظه به دلیل [تکه‌تکه شدن داخلی] هدر می‌رود.

[تخصیص‌دهنده با اندازه بلوک ثابت]: @/edition-2/posts/11-allocator-designs/index.md#fixed-size-block-allocator

طرح‌های تخصیص‌دهنده بسیار بیشتری با مبادلات مختلف وجود دارد. [تخصیص اسلب] برای بهینه‌سازی تخصیص ساختارهای معمولی با اندازه ثابت به خوبی عمل می‌کند، اما در همه موقعیت‌ها قابل اجرا نیست. [تخصیص بادی] از یک درخت باینری برای ادغام بلوک‌های آزاد شده با هم استفاده می‌کند، اما مقدار زیادی از حافظه را هدر می‌دهد زیرا فقط از اندازه‌های بلوک توان ۲ پشتیبانی می‌کند. همچنین مهم است که به یاد داشته باشید که هر پیاده‌سازی هسته دارای حجم کاری منحصر به فرد است، بنابراین _بهترین_ طرح تخصیص‌دهنده وجود ندارد که با همه موارد مطابقت داشته باشد.

[تخصیص اسلب]: @/edition-2/posts/11-allocator-designs/index.md#slab-allocator
[تخصیص بادی]: @/edition-2/posts/11-allocator-designs/index.md#buddy-allocator


## بعدی چیست؟

با این پست، فعلاً پیاده‌سازی مدیریت حافظه خود را به پایان می‌رسانیم. در مرحله بعد، شروع به کاوش در [_چند وظیفه‌ای_] می‌کنیم که با [_نخ‌ها_] شروع می‌شود. در پست بعدی، [_چند پردازشی_]، [_پردازش‌ها_]، و چند وظیفه‌ای تعاونی را در قالب [_async/await_] بررسی خواهیم کرد.

[_چند وظیفه‌ای_]: https://en.wikipedia.org/wiki/Computer_multitasking
[_نخ‌ها_]: https://en.wikipedia.org/wiki/Thread_(computing)
[_پردازش‌ها_]: https://en.wikipedia.org/wiki/Process_(computing)
[_چند پردازشی_]: https://en.wikipedia.org/wiki/Multiprocessing
[_async/await_]: https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html
