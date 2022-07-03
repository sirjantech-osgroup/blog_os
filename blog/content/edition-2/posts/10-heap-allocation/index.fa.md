+++
title = "تخصیص هیپ"
weight = 10
path = "fa/heap-allocation"
date = 2019-06-26

[extra]
chapter = "Memory Management"
# Please update this when updating the translation
translation_based_on_commit = "3e87916b6c2ed792d1bdb8c0947906aef9013ac1"
# GitHub usernames of the people that translated this post
translators = ["hamidrezakp", "MHBahrampour"]
rtl = true
+++

این پست پشتیبانی از تخصیص هیپ (کلمه: heap) را به هسته اضافه می‌کند. در ابتدا، مقدمه‌ای بر حافظه پویا ارائه می‌دهد و نشان می‌دهد که چگونه borrow checker از خطاهای رایج مربوط به تخصیص جلوگیری می‌کند. سپس رابط تخصیص اصلیِ راست را پیاده‌سازی کرده، یک ناحیه حافظه هیپ و یک کِرِیت اختصاص دهنده ایجاد می‌کند. در پایان این پست، همه نوع تخصیص و جمع‌آوری کِرِیت توکار `alloc` برای هسته در دسترس خواهد بود.

<!-- more -->

این بلاگ بصورت آزاد روی [گیت‌هاب] توسعه داده شده است. اگر شما مشکل یا سوالی دارید، لطفاً آن‌جا یک ایشو باز کنید. شما همچنین می‌توانید [در زیر] این پست کامنت بگذارید. منبع کد کامل این پست را می‌توانید در بِرَنچ [`post-10`][post branch] پیدا کنید.

[گیت‌هاب]: https://github.com/phil-opp/blog_os
[در زیر]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-10

<!-- toc -->

## متغیرهای محلی و استاتیک

ما در حال حاضر از دو نوع متغیر در هسته خود استفاده می‌کنیم: متغیرهای محلی و متغیرهای `static`. متغیرهای محلی بر روی [پشته فراخوانی] (به انگلیسی: call stack) ذخیره می‌شوند و فقط تا زمانی‌که تابع محیطی (به انگلیسی: surrounding function) بازگردد، معتبر هستند. متغیرهای استاتیک در یک مکان با حافظه ثابت ذخیره می‌شوند و تا انتهای عمر برنامه زنده هستند.

### متغیرهای محلی

متغیرهای محلی در [پشته فراخوانی] ذخیره می‌شوند، که یک [ساختار داده پشته] (به انگلیسی: stack data structure) است و از عملیات `push` و `pop` پشتیبانی می‌کند. در هر ورودی تابع، پارامترها، آدرس بازگشت و متغیرهای محلی تابع فراخوانی شده توسط کامپایلر push می‌شوند:

[پشته فراخوانی]: https://en.wikipedia.org/wiki/Call_stack
[ساختار داده پشته]: https://en.wikipedia.org/wiki/Stack_(abstract_data_type)

![An outer() and an inner(i: usize) function. Both have some local variables. Outer calls inner(1). The call stack contains the following slots: the local variables of outer, then the argument `i = 1`, then the return address, then the local variables of inner.](call-stack.svg)

مثال بالا پشته فراخوانی را بعد از یک تابع `outer` به نام تابع `inner` نشان می‌دهد. می‌بینیم که پشته فراخوانی ابتدا متغیرهای محلی `external` را شامل می‌شود. در فراخوانی `inner` پارامتر`1` و آدرس بازگشت تابع push شد. سپس کنترل به `inner` منتقل شد که متغیرهای محلی آن را push کرد.

پس از بازگشت تابع `inner`، قسمت پشته فراخوانی دوباره pop می‌شود و فقط متغیرهای محلی `outer` باقی می‌مانند:

![The call stack containing only the local variables of outer](call-stack-return.svg)

می‌بینیم که متغیرهای محلی `inner` فقط تا زمانی که تابع بازگردد، عمر می‌کنند. کامپایلر راست این طول عمرها را اعمال می‌کند و هنگامی که از یک مقدار بیش از حد طولانی استفاده می کنیم، خطا می‌دهد، به عنوان مثال هنگامی که سعی می‌کنیم یک 

 را به یک متغیر محلی برگردانیم:

```rust
fn inner(i: usize) -> &'static u32 {
    let z = [1, 2, 3];
    &z[i]
}
```

([run the example on the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=6186a0f3a54f468e1de8894996d12819))

در حالی که بازگشت مرجع در این مثال منطقی نیست ، مواردی وجود دارد که ما می خواهیم یک متغیر بیشتر از تابع عمر کند. ما قبلاً چنین موردی را در هسته خود مشاهده کردیم هنگامی که سعی کردیم [جدول توصیف کننده وقفه را بارگذاری کنیم] و مجبور شدیم از یک متغیر `static` برای افزایش طول عمر استفاده کنیم.

[جدول توصیف کننده وقفه را بارگذاری کنیم]: @/edition-2/posts/05-cpu-exceptions/index.md#loading-the-idt

### متغیرهای استاتیک

متغیرهای استاتیک در یک مکان با حافظه ثابت، جدا از پشته ذخیره می‌شوند. این مکان حافظه در زمان کامپایل توسط پیوند دهنده تعیین شده و در فایل اجرایی کدگذاری می‌شود. استاتیک‌ها به اندازه مدت زمان اجرای برنامه بصورت کامل، عمر می‌کنند، بنابراین آن‌ها دارای طول عمر `'static` هستند و همیشه می‌توان از متغیرهای محلی به آن‌ها اشاره کرد:

![The same outer/inner example with the difference that inner has a `static Z: [u32; 3] = [1,2,3];` and returns a `&Z[i]` reference](call-stack-static.svg)

وقتی تابع `inner` در مثال بالا باز می‌گردد، بخشی از پشته فراخوانی تخریب می‌شود. متغیرهای استاتیک در یک محدوده حافظه جداگانه قرار دارند که هرگز نابود نمی‌شود، بنابراین مرجع `&Z[1]` هنوز بعد از بازگشت تابع معتبر است.

علاوه بر طول عمر `'static`، متغیرهای استاتیک نیز دارای ویژگی مفیدی هستند که مکان آن‌ها در زمان کامپایل شناخته شده‌است، به طوری که هیچ مرجعی برای دسترسی به آن مورد نیاز نیست. ما از این ویژگی برای ماکروی `println` استفاده کردیم: با استفاده از یک [static `Writer`] داخلی، هیچ مرجع `&mut Writer` وجود ندارد که نیاز به درخواست کردن ماکرو (به انگلیسی: invoke) داشته باشد، که در [کنترل کننده‌های استثنا] (به انگلیسی: exception handlers) بسیار مفید است، جایی که در آن به هیچ متغیر اضافی دسترسی نداریم.

[static `Writer`]: @/edition-2/posts/03-vga-text-buffer/index.md#a-global-interface
[کنترل کننده‌های استثنا]: @/edition-2/posts/05-cpu-exceptions/index.md#implementation

با این حال، این ویژگی متغیرهای استاتیک یک اشکال اساسی دارد: آن‌ها به‌طور پیش‌فرض فقط خواندنی هستند. راست به این دلیل روی این مسئله اصرار دارد چون می‌تواند منجر به رخ دادن [داده رقابتی] (به انگلیسی: data race) شود، به‌عنوان مثال وقتی که دو نخ (به انگلیسی: thread)، یک متغیر استاتیک را به‌طور همزمان تغییر می‌دهند. تنها راهی که می‌توان یک متغیر استاتیک را تغییر داد، محاسبه آن در یک نوع [`Mutex`] است، که تضمین می‌کند که تنها یک مرجع `&mut` در هر نقطه از زمان وجود دارد. ما قبلاً از یک `Mutex` برای [static VGA buffer `Writer`][vga mutex] استفاده کردیم.

[داده رقابتی]: https://doc.rust-lang.org/nomicon/races.html
[`Mutex`]: https://docs.rs/spin/0.5.2/spin/struct.Mutex.html
[vga mutex]: @/edition-2/posts/03-vga-text-buffer/index.md#spinlocks

## حافظه پویا

متغیرهای محلی و استاتیک، درحال‌حاضر در کنار هم بسیار قدرتمند هستند و بیشتر موارد استفاده را پوشش می‌دهند. با این‌حال‌، دیدیم که هر دو محدودیت‌های خود را دارند:

- متغیرهای محلی فقط تا پایان تابع یا بلوکی که در آن هستند زنده می‌مانند. دلیلش این است که آن‌ها روی پشته فراخوانی زندگی می‌کنند و پس از بازگشت تابع از بین می‌روند.

- متغیرهای استاتیک‌ به اندازه مدت زمان اجرای برنامه بصورت کامل، عمر می‌کنند، بنابراین پس از این‌که دیگر استفاده‌ای از آن‌ها نداریم، هیچ راهی برای بازیابی و استفاده مجدد از حافظه آن‌ها وجود ندارد. همچنین، آن‌ها معنی‌شناسی مالکیت نامشخصی دارند و از همه توابع قابل دسترسی هستند، بنابراین هنگامی که می‌خواهیم آن‌ها را اصلاح کنیم، باید با [Mutex] محافظت شوند.

محدودیت دیگر متغیرهای محلی و استاتیک این است که اندازه آن‌ها ثابت است. بنابراین آن‌ها نمی‌توانند مجموعه‌ای را که با افزودن عناصر بیشتر به صورت پویا رشد می‌کند را ذخیره کنند. (البته [unsized rvalues] در راست وجود دارد که متغیرهای محلی با اندازه پویا را تحقق می‌بخشد، اما آن‌ها فقط در برخی موارد خاص کار می‌کنند.)

[unsized rvalues]: https://github.com/rust-lang/rust/issues/48055

برای دور زدن این معایب، زبان‌های برنامه‌نویسی اغلب از یک ناحیه حافظه دیگر برای ذخیره متغیرهایی به نام **heap** پشتیبانی می‌کنند. هیپ از _تخصیص حافظه پویا_ در زمان اجرا از طریق دو تابع به نام‌های `allocate` و `deallocate` پشتیبانی می‌کند. این به روش زیر عمل می کند: تابع `allocate` یک تکه حافظه آزاد با اندازه مشخص را برمی‌گرداند که می‌تواند برای ذخیره یک متغیر استفاده شود. سپس این متغیر زنده می‌ماند تا زمانی که با فراخوانی تابع `deallocate` همراه با یک اشاره‌گر به متغیر، آزاد شود.

بیایید یک مثال را مرور کنیم:

![The inner function calls `allocate(size_of([u32; 3]))`, writes `z.write([1,2,3]);`, and returns `(z as *mut u32).offset(i)`. The outer function does `deallocate(y, size_of(u32))` on the returned value `y`.](call-stack-heap.svg)

در اینجا، تابع `inner` به جای متغیرهای استاتیک، از حافظه هیپ برای ذخیره `z` استفاده می‌کند. ابتدا یک بلوک حافظه با اندازه مورد نیاز را اختصاص می‌دهد، که [اشاره‌گر خام] `*mut u32` را برمی‌گرداند. سپس از متد [`ptr::write`] برای نوشتن آرایه `[1،2،3]` درون آن استفاده می‌کند. در آخرین مرحله، از تابع [`offset`] برای محاسبه اشاره‌گر به `i`مین عنصر استفاده می‌کند و سپس آن را برمی‌گرداند. (توجه داشته باشید که در این تابع مثال، به منظور اختصار، برخی از موارد مورد نیاز و بلوک‌های ناامن را حذف کرده‌ایم.)

[اشاره‌گر خام]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`ptr::write`]: https://doc.rust-lang.org/core/ptr/fn.write.html
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

حافظه تخصیص‌یافته تا زمانی که به صراحت از طریق فراخوانی `deallocate` آزاد نشود، عمر می‌کند. بنابراین‌، اشاره‌گری که بازگشت داده شده، حتی پس از بازگشت تابع `inner` و از بین رفتن قسمت پشته فراخوانی همچنان معتبر است. مزیت استفاده از حافظه هیپ در مقایسه با حافظه استاتیک این است که می‌توان پس از آزادسازی مجدد از حافظه استفاده کرد، که این کار را از طریق فراخوانی `deallocate` در `outer` انجام می‌دهیم. پس از آن فراخوانی، وضعیت به شرح زیر است:

![The call stack contains the local variables of outer, the heap contains z[0] and z[2], but no longer z[1].](call-stack-heap-freed.svg)

می‌بینیم که شکاف `z[1]` دوباره آزاد است و می‌تواند برای فراخوانی بعدی `allocate` مجدداً استفاده شود. با این حال، همچنین می‌بینیم که `z[0]` و `z[2]` هرگز آزاد نمی‌شوند زیرا آن‌ها را هیچ‌گاه تخصیص نداده‌ایم. چنین اشکالی _نشت حافظه_ (به انگلیسی: memory leak) نامیده می‌شود و اغلب، علت مصرف بیش از حد برنامه‌ها از حافظه است (فقط تصور کنید چه اتفاقی می افتد وقتی `inner` را بصورت مداوم در یک حلقه تکرار کنیم). این ممکن است بد به نظر برسد، اما اشکالات بسیار خطرناک‌تری وجود دارد که با تخصیص پویا ممکن است رخ دهد.

### خطاهای رایج

جدا از نشت حافظه، که مایه تأسف است اما برنامه را در برابر مهاجمان آسیب‌پذیر نمی‌کند، دو نوع باگ رایج با پیامدهای جدی‌تر وجود دارد:

- هنگامی که ما به‌طور تصادفی پس از فراخوانی `deallocate` روی یک متغیر، به استفاده از آن ادامه می‌دهیم، به اصطلاح یک آسیب‌پذیری **use-after-free** داریم. چنین باگی باعث رفتارهای نامشخص می‌شود و اغلب می‌تواند توسط هکرها برای اجرای کد دلخواه مورد سوء استفاده قرار گیرد.
- هنگامی که ما به طور تصادفی یک متغیر را دو بار آزاد می‌کنیم، یک آسیب‌پذیری **double-free** داریم. این مسئله به این دلیل مشکل‌ساز است که ممکن است تخصیصِ متفاوت و جدیدی را که پس از اولین فراخوانی `deallocate` در همان نقطه اختصاص داده شده بود، آزاد کند (به‌ عبارتی، اطلاعات جدیدی که در همان نقطه خالی شده ریخته بودیم را مجدداً پاک می‌کند). بنابراین، می‌تواند دوباره منجر به آسیب‌پذیری use-after-free شود.

این نوع آسیب‌پذیری‌ها معمولاً شناخته شده هستند، بنابراین می‌توان انتظار داشت که مردم تا کنون یاد گرفته‌اند که چگونه از آن‌ها جلوگیری کنند. اما نه، چنین آسیب‌پذیری‌هایی هنوز به‌طور مرتب یافت می‌شوند، به عنوان مثال این آسیب‌پذیری اخیر [use-after-free در سیستم‌عامل لینوکس] [آسیب‌پذیری لینوکس] که امکان اجرای دلخواه کد را فراهم می‌آورد. این نشان می‌دهد که حتی بهترین برنامه‌نویسان همواره قادر به مدیریت صحیح حافظه پویا در پروژه‌های پیچیده نیستند.

[آسیب‌پذیری لینوکس]: https://securityboulevard.com/2019/02/linux-use-after-free-vulnerability-found-in-linux-2-6-through-4-20-11/

برای جلوگیری از این مشکلات، بسیاری از زبان‌ها مانند جاوا یا پایتون با استفاده از تکنیکی به نام [_جمع‌آوری زباله_] (به انگلیسی: garbage collection) حافظه پویا را به طور خودکار مدیریت می‌کنند. ایده این است که برنامه‌نویس هرگز `deallocate` را به صورت دستی فرا نمی‌خواند. در عوض، برنامه مرتباً متوقف شده و برای یافتن متغیرهای هیپِ بلا استفاده، شروع به اسکن کردن می‌کند، سپس آن‌ها را به‌طور خودکار آزاد می‌کند. بنابراین‌، آسیب‌پذیری‌های فوق هرگز نمی‌توانند رخ دهند. معایب عبارتند از: سربار عملکرد اسکن کردن مداوم و زمان‌های مکث طولانی.

[_جمع‌آوری زباله_]: https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)

راست رویکرد متفاوتی نسبت به این مشکل دارد: از مفهومی به نام [_ مالکیت_] (به انگلیسی: ownership) استفاده می‌کند که قادر است صحت عملکردهای حافظه پویا را در زمان کامپایل بررسی کند. بنابراین برای اجتناب از آسیب‌پذیری‌های ذکر شده نیازی به جمع‌آوری زباله نیست، این بدان معناست که هیچ سربار اضافی وجود ندارد. مزیت دیگر این رویکرد این است که برنامه‌نویس هنوز کنترل دقیق بر استفاده از حافظه پویا را دارد، درست مانند C یا ++C.

[_ownership_]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html

### تخصیص‌ها در راست

کتابخانه استاندارد راست، به‌جای اجازه‌دادن به برنامه‌نویس برای دستیابی به `allocate` و `deallocate`، انواع انتزاعی را ارائه می‌دهد که به طور ضمنی این توابع را فراخوانی می‌کند. مهم‌ترین نوع آن [**`Box`**] است که انتزاعی برای یک مقدار اختصاص داده شده از هیپ است. این یک تابع سازنده [`Box::new`] را ارائه می‌دهد که یک مقدار را دریافت می‌کند، `allocate` را همراه با اندازه آن مقدار فراخوانی می‌کند و سپس مقدار را به شکاف تازه تخصیص یافته روی هیپ منتقل می‌کند. برای آزادسازی مجدد حافظه پشته، نوع `Box` [خاصیت `Drop`] را پیاده‌سازی می‌کند تا در صورت خارج‌شدن از محدوده `deallocate` را فراخوانی کند:

[**`Box`**]: https://doc.rust-lang.org/std/boxed/index.html
[`Box::new`]: https://doc.rust-lang.org/alloc/boxed/struct.Box.html#method.new
[خاصیت `Drop`]: https://doc.rust-lang.org/book/ch15-03-drop.html

```rust
{
    let z = Box::new([1,2,3]);
    […]
} // z goes out of scope and `deallocate` is called
```

این الگو نام عجیبی دارد [_resource acquisition is initialization: RAII_] (به فارسی: کسب منبع در حال راه‌اندازی است). منشاء آن ++C است، جایی که از آن برای پیاده‌سازی یک نوع انتزاعی مشابه به نام [`std::unique_ptr`] استفاده می‌شود.

[_resource acquisition is initialization: RAII_]: https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization
[`std::unique_ptr`]: https://en.cppreference.com/w/cpp/memory/unique_ptr

چنین نوعی به تنهایی برای جلوگیری از تمام باگ‌های use-after-free کافی نیست، زیرا برنامه‌نویسان می‌توانند پس از خارج‌شدن `Box` از محدود و آزاد کردن شکاف حافظه مربوطه، همچنان به منابع دسترسی داشته باشند:

```rust
let x = {
    let z = Box::new([1,2,3]);
    &z[1]
}; // z goes out of scope and `deallocate` is called
println!("{}", x);
```

اینجاست که مالکیت راست وارد می‌شود. به هر مرجع یک انتزاع [طول عمر] (به انگلیسی: lifetime) اختصاص می‌دهد، که محدوده‌ای است که مرجع در آن معتبر است. در مثال بالا، مرجع `x` از آرایه` z` گرفته شده است‌، بنابراین پس از خارج‌شدن `z` از محدوده، نامعتبر می‌شود. وقتی [مثال بالا را در playground اجرا می کنید][playground-2]، می‌بینید که کامپایلر راست واقعاً یک خطا ایجاد می‌کند:

[lifetime]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html
[playground-2]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=28180d8de7b62c6b4a681a7b1f745a48

```
error[E0597]: `z[_]` does not live long enough
 --> src/main.rs:4:9
  |
2 |     let x = {
  |         - borrow later stored here
3 |         let z = Box::new([1,2,3]);
4 |         &z[1]
  |         ^^^^^ borrowed value does not live long enough
5 |     }; // z goes out of scope and `deallocate` is called
  |     - `z[_]` dropped here while still borrowed
```

اصطلاحات می‌تواند در ابتدا کمی گیج‌کننده باشد. به اختصاص دادن یک مرجع به یک مقدار، _borrowing_ گفته می‌شود، زیرا شبیه قرض‌گرفتن (به انگلیسی: borrowing) در زندگی واقعی است: شما به‌طور موقت به یک شی دسترسی دارید، اما باید آن را در نهایت پس دهید و نباید آن را از بین ببرید. با بررسی این‌که تمام قرض‌ها قبل از تخریب یک شیء به پایان می‌رسند، کامپایلر راست می‌تواند تضمین کند که به باگ use-after-free برنخوریم.

سیستم مالکیت راست حتی پا را فراتر می‌گذارد و نه تنها از باگ‌های use-after-free جلوگیری نمی‌کند، بلکه مانند زبان‌های با قابلیت جمع‌آوری زباله، مانند جاوا یا پایتون، [_ایمنی حافظه_] (به انگلیسی: memory safety) را به‌صورت کامل فراهم می‌کند. علاوه‌بر این، [_ایمنی نخ_] (به انگلیسی: thread safety) را تضمین می‌کند و بنابراین در کدهای چند نخی، حتی از آن زبان‌ها ایمن‌تر است. و از همه مهم‌تر، همه این بررسی‌ها در زمان کامپایل انجام می‌شود، بنابراین هیچ هزینه سرباری در زمان اجرا، در مقایسه با مدیریت حافظه دستی در C وجود ندارد.

[_ایمنی حافظه_]: https://en.wikipedia.org/wiki/Memory_safety
[_ایمنی نخ_]: https://en.wikipedia.org/wiki/Thread_safety

### موارد استفاده

اکنون اصول تخصیص حافظه پویا در راست را می‌دانیم، اما چه زمانی باید از آن استفاده کنیم؟ ما بدون تخصیص حافظه پویا برای هسته تا اینجا پیش آمدیم، پس چرا حالا به آن نیاز داریم؟

مورد اول این‌که، تخصیص حافظه پویا همیشه با کمی سربار همراه است، زیرا باید برای هر تخصیص یک شکاف آزاد در هیپ پیدا کنیم. به همین دلیل متغیرهای محلی عموماً ترجیح داده می‌شوند، به ویژه در کدِ هسته که کارایی آن بسیار مهم است. با این حال، مواردی وجود دارد که تخصیص حافظه پویا بهترین انتخاب است.

به عنوان یک قاعده اساسی، حافظه پویا برای متغیرهایی که دارای طول عمر پویا یا اندازه متغیر هستند، مورد نیاز است. مهم‌ترین نوع با طول عمر پویا [**`Rc`**] است، که ارجاعات به مقدار بسته‌بندی‌شده (به انگلیسی: wrapped) آن را شمارش می‌کند و پس از خارج شدن همه مراجع از محدوده، آن را آزاد می‌کند. مثال‌های دیگر برای انواع با اندازه متغیر عبارتند از [**`Vec`**] ، [**`String`**] و دیگر [انواع مجموعه] (به انگلیسی: collection types)که با افزودن عناصر بیشتر به صورت پویا رشد می‌کنند. نحوه کار این انواع به این‌صورت است که هنگام پر شدن حافظه فعلی، یک حافظه بزرگ‌تر اختصاص می‌دهند، تمام عناصر فعلی را کپی کرده و حافظه قبلی را آزاد می‌کنند.

[**`Rc`**]: https://doc.rust-lang.org/alloc/rc/index.html
[**`Vec`**]: https://doc.rust-lang.org/alloc/vec/index.html
[**`String`**]: https://doc.rust-lang.org/alloc/string/index.html
[انواع مجموعه]: https://doc.rust-lang.org/alloc/collections/index.html

برای هسته ما، بیشتر به انواع مجموعه نیاز داریم، به‌ عنوان مثال برای ذخیره لیستی از وظایف فعال، هنگام پیاده‌سازی چند وظیفگی (به انگلیسی: multitasking) در پست‌های بعدی.

## رابط تخصیص‌دهنده

اولین قدم برای پیاده‌سازی یک تخصیص دهنده هیپ، افزودن وابستگی به کِرِیت [`alloc`] توکار است. مانند کِرِیت ['core']، زیر مجموعه‌ای از کتابخانه استاندارد است که علاوه بر آن شامل انواع allocation و collection است. برای افزودن وابستگی به `alloc` موارد زیر را به `lib.rs` اضافه می‌کنیم:

[`alloc`]: https://doc.rust-lang.org/alloc/
[`core`]: https://doc.rust-lang.org/core/

```rust
// in src/lib.rs

extern crate alloc;
```

برخلاف وابستگی‌های معمولی، نیازی به تغییر `Cargo.toml` نداریم. دلیل این امر این است که کِرِیت `alloc` با کامپایلر راست به عنوان بخشی از کتابخانه استاندارد به سیستم‌عاملی که راست را روی آن نصب می‌کنید، ارسال می‌شود، بنابراین کامپایلر در حال حاضر از کِرِیت مورد نظر اطلاع دارد. با افزودن عبارت `extern crate`، مشخص می‌کنیم که کامپایلر باید سعی کند آن را به کد اضافه کند. (از لحاظ تاریخی، همه وابستگی‌ها به یک عبارت `extern crate` نیاز داشتند، که اکنون اختیاری است).

از آن‌جایی که ما برای یک هدف سفارشی در حال کامپایل کردن هستیم، نمی‌توانیم از نسخه پیش‌کامپایل شده `alloc` استفاده کنیم که با نصب راست ارسال می‌شود. در عوض، ما باید به کارگو بگوییم تا کِرِیت را از منبع دوباره کامپایل کند. می‌توانیم این کار را با افزودن آن به آرایه `unstable.build-std` در فایل` .cargo/config.toml` انجام دهیم:

```toml
# in .cargo/config.toml

[unstable]
build-std = ["core", "compiler_builtins", "alloc"]
````

Now the compiler will recompile and include the `alloc` crate in our kernel.

The reason that the `alloc` crate is disabled by default in `#[no_std]` crates is that it has additional requirements. We can see these requirements as errors when we try to compile our project now:

```
error: no global memory allocator found but one is required; link to std or add
       #[global_allocator] to a static item that implements the GlobalAlloc trait.

error: `#[alloc_error_handler]` function required, but not found
```

The first error occurs because the `alloc` crate requires an heap allocator, which is an object that provides the `allocate` and `deallocate` functions. In Rust, heap allocators are described by the [`GlobalAlloc`] trait, which is mentioned in the error message. To set the heap allocator for the crate, the `#[global_allocator]` attribute must be applied to a `static` variable that implements the `GlobalAlloc` trait.

The second error occurs because calls to `allocate` can fail, most commonly when there is no more memory available. Our program must be able to react to this case, which is what the `#[alloc_error_handler]` function is for.

[`GlobalAlloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html

We will describe these traits and attributes in detail in the following sections.

### The `GlobalAlloc` Trait

The [`GlobalAlloc`] trait defines the functions that a heap allocator must provide. The trait is special because it is almost never used directly by the programmer. Instead, the compiler will automatically insert the appropriate calls to the trait methods when using the allocation and collection types of `alloc`.

Since we will need to implement the trait for all our allocator types, it is worth taking a closer look at its declaration:

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

این دو متد مورد نیازِ [`alloc`] و [`dealloc`] را تعریف می‌کند که مربوط به توابع `allocate` و `deallocate` هستند که در مثال‌ها خود استفاده کردیم:

- متد [`alloc`] یک نمونه [`Layout`] را به عنوان آرگومان می‌گیرد، که اندازه و تراز مورد نظر را که حافظه اختصاص داده شده باید داشته باشد را توصیف می‌کند. این یک [اشاره‌گر خام] را به اولین بایت بلوک حافظه‌ی اختصاص داده شده برمی‌گرداند. به جای مقدار خطای صریح، متد `alloc`، یک اشاره‌گر null برمی‌گرداند تا با آن خطای تخصیص را نشان دهد. این کمی غیر اصطلاحی است، اما این مزیت را دارد که بسته‌بندیِ تخصیص‌دهندگانِ موجودِ سیستم آسان است، زیرا آن‌ها از یک قرارداد استفاده می‌کنند.
- متد [`dealloc`] همتاست و مسئول آزادسازی مجدد یک بلوک حافظه است. دو آرگومان دریافت می‌کند، اشاره‌گر برگشته از `alloc` و نمونه‌ای از `Layout` که برای تخصیص استفاده شده بود.

[`alloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.alloc
[`dealloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.dealloc
[`Layout`]: https://doc.rust-lang.org/alloc/alloc/struct.Layout.html

این خاصیت علاوه‌بر موارد قبل، دو متدِ [`alloc_zeroed`] و [`realloc`] را با پیاده‌سازی‌های پیش‌فرض تعریف می‌کند:

- متد [`alloc_zeroed`] معادل فراخوانی `alloc` و سپس تنظیم بلوک حافظه اختصاص داده‌ شده بر روی صفر است، که دقیقاً همان کاری است که پیاده‌سازیِ پیش‌فرضِ ارائه‌شده انجام می‌دهد. پیاده‌سازیِ تخصیص‌دهنده می‌تواند در صورت امکان، پیاده‌سازی‌های پیش‌فرض را با پیاده‌سازی سفارشیِ کارآمدتر، بازنویسی کند.
- متد [`realloc`] امکان افزایش‌دادن یا کوچک‌کردن تخصیص را می‌دهد. پیاده‌سازی پیش‌فرض یک بلوک حافظه جدید با اندازه دلخواه اختصاص می‌دهد و تمام محتویات مربوط به تخصیص قبلی را کپی می‌کند. باز هم، پیاده‌سازی تخصیص‌دهنده احتمالاً می‌تواند پیاده‌سازی کارآمدتری از این روش را ارائه دهد، به عنوان مثال، در صورت امکان، با افزایش‌دادن یا کوچک‌کردن تخصیص، در همان محل.

[`alloc_zeroed`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#method.alloc_zeroed
[`realloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#method.realloc

#### ناامنی

نکته‌ای که باید به آن توجه کرد این است که هم خود خاصیت (به انگلیسی: trait) و هم همه متدهای آن به عنوان `unsafe` اعلام می‌شوند:

- دلیل اعلام خاصیت به عنوان `unsafe` این است که برنامه‌نویس باید تضمین کند که پیاده‌سازی خاصیت برای یک نوع تخصیص‌دهنده صحیح است. به عنوان مثال، متد `alloc` هرگز نباید یک بلوک حافظه را که قبلاً در جای دیگری استفاده شده است برگرداند زیرا این امر باعث رفتار نامشخص می شود.
- به طور مشابه، دلیل `unsafe` بودن متدها این است که فراخواننده (به انگلیسی: caller) باید در هنگام فراخوانی متدها از متغیرهای مختلف اطمینان حاصل کند، به عنوان مثال `Layout` که به `alloc` منتقل شده است، یک اندازه غیر صفر را مشخص می‌کند. چنین چیزی در عمل واقعاً مرتبط نیست زیرا معمولاً متدها مستقیماً توسط کامپایلر فراخوانی می‌شوند که تضمین می‌کند الزامات برآورده شوند.

### یک `DummyAllocator`

اکنون که می‌دانیم یک نوع تخصیص‌دهنده چه چیزی را باید ارائه دهد، می‌توانیم یک تخصیص‌دهنده ساختگی ساده ایجاد کنیم. برای این کار یک ماژول `allocator` جدید ایجاد می‌کنیم:

```rust
// in src/lib.rs

pub mod allocator;
```

تخصیص‌دهنده ساختگی ما کمترین کار ممکن را برای پیاده‌سازی این خاصیت انجام می‌دهد و همیشه هنگام فراخوانی `alloc` یک خطا برمی‌گرداند. کد آن چیزی مشابه این است:

```rust
// in src/allocator.rs

use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr::null_mut;

pub struct Dummy;

unsafe impl GlobalAlloc for Dummy {
    unsafe fn alloc(&self, _layout: Layout) -> *mut u8 {
        null_mut()
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        panic!("dealloc should be never called")
    }
}
```

ساختار (به انگلیسی: struct) به هیچ فیلدی نیاز ندارد، بنابراین آن را به صورت [zero sized type] ایجاد می‌کنیم. همان‌طور که در بالا ذکر شد، ما همیشه اشاره‌گر تهی را از `alloc` برمی‌گردانیم که یک خطای تخصیص را بوجود می‌آورد. از آن‌جایی که تخصیص دهنده هرگز هیچ حافظه‌ای را بر نمی‌گرداند، هرگز نباید `dealloc` را فراخوانی کرد. به همین دلیل به سادگی در متد `dealloc` یک پنیک (کلمه: panic) ایجاد می‌کنیم. متدهای `alloc_zeroed` و `realloc` پیاده‌سازی‌های پیش‌فرض دارند، بنابراین نیازی به ارائه پیاده‌سازی برای آن‌ها نداریم.

[zero sized type]: https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts

ما اکنون یک تخصیص دهنده ساده داریم، اما همچنان باید به کامپایلر راست بگوییم که باید از این تخصیص‌دهنده استفاده کند. اینجاست که ویژگی `#[global_allocator]` وارد می‌شود.

### صفت `#[global_allocator]`

صفت `#[global_allocator]` به کامپایلر راست می‌گوید که از کدام نمونه تخصیص‌دهنده به عنوان تخصیص‌دهنده هیپ عمومی استفاده کند. این ویژگی فقط برای یک `static` که خاصیت `GlobalAlloc` را پیاده‌سازی می‌کند قابل استفاده است. بیایید نمونه‌ای از تخصیص‌دهنده `Dummy` را به عنوان تخصیص‌دهنده عمومی ثبت کنیم:

```rust
// in src/allocator.rs

#[global_allocator]
static ALLOCATOR: Dummy = Dummy;
```

از آنجایی که تخصیص دهنده `Dummy` یک [zero sized type] است، نیازی به تعیین هیچ فیلدی در عبارت اولیه نداریم.

اگر حالا آن را کامپایل کنیم، اولین خطا باید از بین برود. بیایید خطای دومین خطای باقی‌مانده را برطرف کنیم:

```
error: `#[alloc_error_handler]` function required, but not found
```

### صفت `#[alloc_error_handler]`

همانطور که در هنگام بحث در مورد خاصیت `GlobalAlloc` یاد گرفتیم، تابع `alloc` می‌تواند با برگرداندن یک اشاره‌گر تهی، یک خطای تخصیص را سیگنال کند. سوال این است: راست در زمان اجرا چگونه باید به چنین شکست تخصیصی واکنش نشان دهد؟ اینجاست که صفت `#[alloc_error_handler]` وارد عمل می‌شود. این صفت تابعی را مشخص می‌کند که هنگام وقوع خطای تخصیص فراخوانی می‌شود.

بیایید چنین تابعی را برای رفع خطای کامپایل اضافه کنیم:

```rust
// in src/lib.rs

#![feature(alloc_error_handler)] // at the top of the file

#[alloc_error_handler]
fn alloc_error_handler(layout: alloc::alloc::Layout) -> ! {
    panic!("allocation error: {:?}", layout)
}
```

تابع `alloc_error_handler` هنوز ناپایدار است، بنابراین برای فعال کردن آن به یک دروازه ویژگی (به انگلیسی: feature gate) نیاز داریم. تابع یک آرگومان واحد دریافت می‌کند: نمونه `Layout` که در زمان وقوع شکست تخصیص به `alloc` ارسال شد. هیچ کاری نمی‌توانیم برای رفع این مشکل انجام دهیم، بنابراین فقط با پیامی که حاوی نمونه `Layout` است پَنیک می‌کنیم.

با این تابع، خطاهای کامپایل باید رفع شوند. اکنون می‌توانیم از تخصیص و انواع مجموعه `alloc` استفاده کنیم، برای مثال می‌توانیم از یک [`Box`] برای تخصیص یک مقدار روی هیپ استفاده کنیم:

[`Box`]: https://doc.rust-lang.org/alloc/boxed/struct.Box.html

```rust
// in src/main.rs

extern crate alloc;

use alloc::boxed::Box;

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] print "Hello World!", call `init`, create `mapper` and `frame_allocator`

    let x = Box::new(41);

    // […] call `test_main` in test mode

    println!("It did not crash!");
    blog_os::hlt_loop();
}

```

توجه داشته باشید که ما باید عبارت `extern crate alloc` را در `main.rs` نیز مشخص کنیم. این مورد ضروری است زیرا بخش `lib.rs` و `main.rs` به عنوان کریت‌های جداگانه در نظر گرفته می‌شوند. با این حال، ما نیازی به ایجاد یک `#[global_allocator]` ثابتِ دیگر نداریم زیرا تخصیص‌دهنده سراسری برای همه کریت‌های پروژه اعمال می‌شود. در واقع، تعیین یک تخصیص دهنده اضافی در یک کریت دیگر باعث ایجاد خطا می‌شود.

وقتی کد بالا را اجرا می‌کنیم، می‌بینیم که تابع `alloc_error_handler` فراخوانی می‌شود:

![QEMU printing "panicked at `allocation error: Layout { size_: 4, align_: 4 }, src/lib.rs:89:5"](qemu-dummy-output.png)

کنترل‌کننده خطا به این دلیل فراخوانی می‌شود که تابع `Box::new` به طور ضمنی تابع `alloc` تخصیص‌دهنده سراسری را فراخوانی می‌کند. تخصیص‌دهنده ساختگی ما همیشه یک اشاره‌گر تهی را برمی‌گرداند، بنابراین هر تخصیص با شکست مواجه می‌شود. برای رفع این مشکل، باید یک تخصیص‌دهنده ایجاد کنیم که در واقع یک حافظه قابل استفاده را برمی‌گرداند.

## ساخت یک هیپِ هسته

قبل از اینکه بتوانیم یک تخصیص‌دهنده مناسب ایجاد کنیم، ابتدا باید یک ناحیه حافظه هیپ ایجاد کنیم که تخصیص‌دهنده بتواند حافظه را از آن اختصاص دهد. برای این کار باید یک محدوده حافظه مجازی برای ناحیه هیپ تعریف کنیم و سپس این ناحیه را به قاب‌های فیزیکی نگاشت کنیم. برای مروری بر حافظه مجازی و جداول صفحه، پست [_"مقدمه‌ای بر صفحه‌بندی"_] را ببینید.

[_"مقدمه‌ای بر صفحه‌بندی"_]: @/edition-2/posts/08-paging-introduction/index.fa.md

اولین قدم این است که یک محدوده حافظه مجازی برای هیپ تعریف کنیم. می‌توانیم هر محدوده آدرس مجازی را که دوست داریم انتخاب کنیم، به شرطی که قبلاً برای یک ناحیه حافظه متفاوت استفاده نشده باشد. بیایید آن را به‌عنوان حافظه‌ای تعریف کنیم که از آدرس `0x_4444_4444_0000` شروع می‌شود تا بعداً بتوانیم اشاره‌گر پشته را به راحتی تشخیص دهیم:

```rust
// in src/allocator.rs

pub const HEAP_START: usize = 0x_4444_4444_0000;
pub const HEAP_SIZE: usize = 100 * 1024; // 100 KiB
```

اندازه پشته را فعلاً روی 100 کیلوبایت تنظیم کردیم. اگر در آینده به فضای بیشتری نیاز داشته باشیم، می‌توانیم به سادگی آن را افزایش دهیم.

اگر اکنون بخواهیم از این ناحه هیپ استفاده کنیم، یک خطای‌صفحه رخ می‌دهد زیرا ناحیه حافظه‌مجازی هنوز به حافظه‌فیزیکی نگاشت نشده است. برای حل این مشکل، یک تابع `init_heap` ایجاد می‌کنیم که صفحات پشته را با استفاده از [`Mapper` API] که در پست [_"پیاده‌سازی صفحه‌بندی"_] معرفی کردیم، نگاشت می‌کند:

[`Mapper` API]: @/edition-2/posts/09-paging-implementation/index.md#using-offsetpagetable
[_"پیاده‌سازی صفحه‌بندی"_]: @/edition-2/posts/09-paging-implementation/index.md

```rust
// in src/allocator.rs

use x86_64::{
    structures::paging::{
        mapper::MapToError, FrameAllocator, Mapper, Page, PageTableFlags, Size4KiB,
    },
    VirtAddr,
};

pub fn init_heap(
    mapper: &mut impl Mapper<Size4KiB>,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) -> Result<(), MapToError<Size4KiB>> {
    let page_range = {
        let heap_start = VirtAddr::new(HEAP_START as u64);
        let heap_end = heap_start + HEAP_SIZE - 1u64;
        let heap_start_page = Page::containing_address(heap_start);
        let heap_end_page = Page::containing_address(heap_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };

    for page in page_range {
        let frame = frame_allocator
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        unsafe {
            mapper.map_to(page, frame, flags, frame_allocator)?.flush()
        };
    }

    Ok(())
}
```

این تابع ارجاعات قابل‌تغییر به یک نمونه [`Mapper`] و [`FrameAllocator`] می‌گیرد که هر دو با استفاده از [`Size4KiB`] به عنوان پارامتر عمومی به صفحات 4 کیلوبایتی محدود می‌شوند. مقدار بازگشتی تابع یک [`Result`] با نوع واحد `()` به عنوان حالت موفقیت و یک [`MapToError`] به عنوان حالت خطا است، که نوع خطای بازگشتی توسط متد [`Mapper::map_to`] است. استفاده مجدد از نوع خطا در اینجا منطقی است زیرا روش `map_to` منبع اصلی خطاها در این تابع است.

[`Mapper`]:https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/trait.Mapper.html
[`FrameAllocator`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/trait.FrameAllocator.html
[`Size4KiB`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/page/enum.Size4KiB.html
[`Result`]: https://doc.rust-lang.org/core/result/enum.Result.html
[`MapToError`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/enum.MapToError.html
[`Mapper::map_to`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/trait.Mapper.html#method.map_to

پیاده‌سازی را می‌توان به دو بخش تقسیم کرد:

- **ایجاد محدوده صفحه:** برای ایجاد محدوده‌ای از صفحاتی که می‌خواهیم نگاشت کنیم، اشاره‌گر `HEAP_START` را به نوع [`VirtAddr`] تبدیل می‌کنیم. سپس با افزودن `HEAP_SIZE` آدرس انتهای پشته را از روی آن محاسبه می‌کنیم. ما یک حدِ شامل (به انگلیسی: inclusive bound) (آدرس آخرین بایت هیپ) می‌خواهیم، بنابراین منهای ۱ می‌کنیم. سپس، آدرس‌ها را با استفاده از تابع [`containing_address`] به انواع [`Page`] تبدیل می‌کنیم. در نهایت، با استفاده از تابع [`Page::range_inclusive`] یک محدوده صفحه از صفحات شروع و پایان ایجاد می‌کنیم.

- **نگاشت صفحات:** مرحله دوم نگاشت تمام صفحات محدوده صفحه‌ای است که ایجاد کردیم. برای این کار، با استفاده از یک حلقه `for` روی صفحات آن محدوده پیمایش می‌کنیم. برای هر صفحه، موارد زیر را انجام می‌دهیم:

    - ما یک قاب فیزیکی را اختصاص می‌دهیم که صفحه باید با استفاده از متد [`FrameAllocator::allocate_frame`] به آن نگاشت شود. این متد زمانی که قاب دیگری باقی نمانده است [`None`] را برمی‌گرداند. ما با نگاشت آن به یک خطای [`MapToError::FrameAllocationFailed`] از طریق متد [`Option::ok_or`] با آن برخورد می‌کنیم و سپس از [عملگر علامت سوال] برای بازگشت زودهنگام در صورت بروز خطا استفاده می‌کنیم.

    - ما پرچم `PRESENT` مورد نیاز و پرچم `WRITABLE` را برای صفحه تنظیم می‌کنیم. با وجود این پرچم‌ها دسترسی خواندن و نوشتن مجاز است که برای حافظه هیپ منطقی است.

    - برای ایجاد نگاشت در جدول صفحه فعال از متد [`Mapper::map_to`] استفاده می‌کنیم. این متد ممکن است شکست بخورد، بنابراین ما دوباره از [عملگر علامت سوال] برای ارسال خطا به صدا زننده استفاده می‌کنیم. در صورت موفقیت، متد یک نمونه [`MapperFlush`] را برمی‌گرداند که می‌توانیم از آن برای به روز رسانی [_translation lookaside buffer_] با استفاده از متد [`flush`] استفاده کنیم.

[`VirtAddr`]: https://docs.rs/x86_64/0.12.1/x86_64/addr/struct.VirtAddr.html
[`Page`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/page/struct.Page.html
[`containing_address`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/page/struct.Page.html#method.containing_address
[`Page::range_inclusive`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/page/struct.Page.html#method.range_inclusive
[`FrameAllocator::allocate_frame`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/trait.FrameAllocator.html#tymethod.allocate_frame
[`None`]: https://doc.rust-lang.org/core/option/enum.Option.html#variant.None
[`MapToError::FrameAllocationFailed`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/enum.MapToError.html#variant.FrameAllocationFailed
[`Option::ok_or`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.ok_or
[عملگر علامت سوال]: https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html
[`MapperFlush`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/struct.MapperFlush.html
[_translation lookaside buffer_]: @/edition-2/posts/08-paging-introduction/index.md#the-translation-lookaside-buffer
[`flush`]: https://docs.rs/x86_64/0.12.1/x86_64/structures/paging/mapper/struct.MapperFlush.html#method.flush

آخرین مرحله، فراخوانی این تابع از `kernel_main` است:
The final step is to call this function from our `kernel_main`:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::allocator; // new import
    use blog_os::memory::{self, BootInfoFrameAllocator};

    println!("Hello World{}", "!");
    blog_os::init();

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };

    // new
    allocator::init_heap(&mut mapper, &mut frame_allocator)
        .expect("heap initialization failed");

    let x = Box::new(41);

    // […] call `test_main` in test mode

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

کل تابع را در اینجا نشان می‌دهیم. تنها خطوط جدید وارد کردن `blog_os::allocator` و فراخوانی تابع `allocator::init_heap` است. در صورتی که تابع `init_heap` خطایی را برگرداند، با استفاده از متد [`Result::expect`] پنیک می‌کنیم زیرا در حال حاضر هیچ راه معقولی برای مدیریت این خطا برای ما وجود ندارد.

[`Result::expect`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.expect

اکنون یک محدوده حافظه هیپ نگاشت شده داریم که آماده استفاده است. فراخوانی `Box::new` همچنان از تخصیص‌دهنده قدیمی `Dummy` استفاده می‌کند، بنابراین هنگام اجرای آن همچنان خطای "out of memory" را خواهید دید. بیایید با استفاده از یک تخصیص‌دهنده مناسب این مشکل را برطرف کنیم.

## استفاده از یک کریت تخصیص‌دهنده

از آن‌جایی که پیاده‌سازی یک تخصیص‌دهنده تا حدودی پیچیده است، ما با استفاده از یک کریت تخصیص‌دهنده از پیش ساخته شده شروع می‌کنیم. البته در پست بعد یاد می‌گیرید که چطور یک تخصیص‌دهنده پیاده‌سازی کنید.

یک کریت تخصیص‌دهنده ساده برای برنامه‌های `no_std` کریت [`linked_list_allocator`] است. نام آن از این واقعیت ناشی می‌شود که از یک ساختار داده لیست پیوندی برای پیگیری محدوده حافظه اختصاص داده شده استفاده می‌کند. برای توضیح بیشتر این روش به پست بعدی مراجعه کنید.

برای استفاده از کریت، ابتدا باید یک وابستگی به آن در `Cargo.toml` اضافه کنیم:

[`linked_list_allocator`]: https://github.com/phil-opp/linked-list-allocator/

```toml
# in Cargo.toml

[dependencies]
linked_list_allocator = "0.8.0"
```

سپس می‌توانیم تخصیص‌دهنده ساختگی خود را با تخصیص‌دهنده ارائه شده توسط کریت جایگزین کنیم:

```rust
// in src/allocator.rs

use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();
```

این ساختار `LockedHeap` نام دارد زیرا از نوع [`spinning_top::Spinlock`] برای همگام‌سازی استفاده می‌کند. این مورد ضروری است زیرا چندین رشته می‌توانند به طور همزمان به استاتیک `ALLOCATOR` دسترسی داشته باشند. مثل همیشه هنگام استفاده از spinlock یا mutex، باید مراقب باشیم که تصادفاً بن‌بست ایجاد نکنیم. این بدان معنی است که ما نباید هیچ تخصیصی را در کنترل‌کننده‌های وقفه انجام دهیم، زیرا آن‌ها می‌توانند در یک زمان دلخواه اجرا شوند و ممکن است تخصیصِ در حال انجام را قطع کنند.

[`spinning_top::Spinlock`]: https://docs.rs/spinning_top/0.1.0/spinning_top/type.Spinlock.html

تنظیم `LockedHeap` به عنوان تخصیص‌دهنده سراسری کافی نیست. چرا که ما از تابع سازنده [`empty`] استفاده می‌کنیم که یک تخصیص‌دهنده بدون هیچ حافظه پشتیبان ایجاد می‌کند. مانند تخصیص‌دهنده ساختگی ما، همیشه یک خطا را در `alloc` برمی‌گرداند. برای رفع این مشکل، باید پس از ایجاد هیپ، تخصیص‌دهنده را مقداردهی اولیه کنیم:

[`empty`]: https://docs.rs/linked_list_allocator/0.8.0/linked_list_allocator/struct.LockedHeap.html#method.empty

```rust
// in src/allocator.rs

pub fn init_heap(
    mapper: &mut impl Mapper<Size4KiB>,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) -> Result<(), MapToError<Size4KiB>> {
    // […] map all heap pages to physical frames

    // new
    unsafe {
        ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE);
    }

    Ok(())
}
```

ما از متد [`lock`] در اسپین‌لاکِ (کلمه: spinlock) داخلی از نوع `LockedHeap` استفاده می‌کنیم تا یک مرجع انحصاری به نمونه [`Heap`] دریافت کنیم، که در آن متد [`init`] را با کران هیپ فراخوانی می‌کنیم. مهم است که _پس از_ نگاشت صفحات هیپ، هیپ را مقداردهی اولیه کنیم، زیرا تابع [`init`] از قبل سعی می‌کند در حافظه هیپ بنویسد.

[`lock`]: https://docs.rs/lock_api/0.3.3/lock_api/struct.Mutex.html#method.lock
[`Heap`]: https://docs.rs/linked_list_allocator/0.8.0/linked_list_allocator/struct.Heap.html
[`init`]: https://docs.rs/linked_list_allocator/0.8.0/linked_list_allocator/struct.Heap.html#method.init

پس از مقداردهی اولیه هیپ، اکنون می‌توانیم از همه انواع تخصیص و جمع‌آوری کریت داخلی [`alloc`] بدون خطا استفاده کنیم:

```rust
// in src/main.rs

use alloc::{boxed::Box, vec, vec::Vec, rc::Rc};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialize interrupts, mapper, frame_allocator, heap

    // allocate a number on the heap
    let heap_value = Box::new(41);
    println!("heap_value at {:p}", heap_value);

    // create a dynamically sized vector
    let mut vec = Vec::new();
    for i in 0..500 {
        vec.push(i);
    }
    println!("vec at {:p}", vec.as_slice());

    // create a reference counted vector -> will be freed when count reaches 0
    let reference_counted = Rc::new(vec![1, 2, 3]);
    let cloned_reference = reference_counted.clone();
    println!("current reference count is {}", Rc::strong_count(&cloned_reference));
    core::mem::drop(reference_counted);
    println!("reference count is {} now", Rc::strong_count(&cloned_reference));

    // […] call `test_main` in test context
    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

این مثال، برخی از کاربردهای انواع [`Box`]، [`Vec`] و [`Rc`] را نشان می‌دهد. برای انواع `Box` و `Vec`، اشاره‌گرهای هیپ زیرین را با استفاده از [مشخص‌کننده فرمت‌بندی `{:p}`] چاپ می‌کنیم. برای نمایش `Rc`، یک مقدار هیپ شمارش مرجع ایجاد می‌کنیم و از تابع [`Rc::strong_count`] برای چاپ تعداد مرجع فعلی، قبل و بعد از حذف یک نمونه استفاده می‌کنیم (با استفاده از [`core::mem::drop`]).

[`Vec`]: https://doc.rust-lang.org/alloc/vec/
[`Rc`]: https://doc.rust-lang.org/alloc/rc/
[مشخص‌کننده فرمت‌بندی `{:p}`]: https://doc.rust-lang.org/core/fmt/trait.Pointer.html
[`Rc::strong_count`]: https://doc.rust-lang.org/alloc/rc/struct.Rc.html#method.strong_count
[`core::mem::drop`]: https://doc.rust-lang.org/core/mem/fn.drop.html

وقتی آن را اجرا کنیم، خروجی زیر را می‌بینیم:

![QEMU printing `
heap_value at 0x444444440000
vec at 0x4444444408000
current reference count is 2
reference count is 1 now
](qemu-alloc-showcase.png)

همانطور که انتظار می‌رفت، می‌بینیم که مقادیر `Box` و `Vec` روی هیپ وجود دارند، همانطور که با اشاره‌گری که با پیشوند `0x_4444_4444_*` نشان داده می‌شود. مقدار شمارش مرجع نیز همانطور که انتظار می‌رفت رفتار می‌کند، به این‌صورت که مقدار آن پس از فراخوانی `clone` برابر با ۲ و دوباره پس از حذف یکی از نمونه‌ها برابر با ۱ می‌شود.

دلیل این‌که وکتور با آفست `0x800` شروع می‌شود این نیست که مقدار درون باکس `0x800` بایت بزرگ است، بلکه [تخصیص‌های مجددهایی] است که زمانی رخ می‌دهد که وکتور نیاز به افزایش ظرفیت خود دارد. به عنوان مثال، زمانی که ظرفیت وکتور ۳۲ است و ما سعی می‌کنیم عنصر بعدی را اضافه کنیم، وکتور یک آرایه پشتیبان جدید با ظرفیت ۶۴ در پس‌زمینه اختصاص می‌دهد و همه عناصر را کپی می‌کند. سپس تخصیص قدیمی را آزاد می‌کند.

[تخصیص‌های مجدد]: https://doc.rust-lang.org/alloc/vec/struct.Vec.html#capacity-and-reallocation

البته انواع تخصیص و مجموعه بسیار بیشتری در کریت `alloc` وجود دارد که اکنون همه می‌توانیم در هسته خود از آن‌ها استفاده کنیم، از جمله:

- اشاره‌گر [`Arc`] شمارش مرجع نخ-امن
- نوع رشته [`String`] و ماکرو [`format!`]
- [`Linked List`]
- بافر حلقه قابل رشد ['VecDeque']
- صف اولویت ['BinaryHeap']
- [`BTreeMap`] و [`BTreeSet`]

[`Arc`]: https://doc.rust-lang.org/alloc/sync/struct.Arc.html
[`String`]: https://doc.rust-lang.org/alloc/string/struct.String.html
[`format!`]: https://doc.rust-lang.org/alloc/macro.format.html
[`LinkedList`]: https://doc.rust-lang.org/alloc/collections/linked_list/struct.LinkedList.html
[`VecDeque`]: https://doc.rust-lang.org/alloc/collections/vec_deque/struct.VecDeque.html
[`BinaryHeap`]: https://doc.rust-lang.org/alloc/collections/binary_heap/struct.BinaryHeap.html
[`BTreeMap`]: https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html
[`BTreeSet`]: https://doc.rust-lang.org/alloc/collections/btree_set/struct.BTreeSet.html

این نوع‌ها زمانی بسیار مفید خواهند بود که بخواهیم لیست‌های نخ، صف‌های زمان‌بندی یا پشتیبانی از async/wait را پیاده‌سازی کنیم.

## اضافه کردن یک تست

برای اطمینان از این‌که به‌طور تصادفی کد تخصیص جدید خود را نمی‌شکنیم، باید یک تست یکپارچه‌سازی برای آن اضافه کنیم. با ایجاد یک فایل `tests/heap_allocation.rs` جدید با محتوای زیر شروع می‌کنیم:

```rust
// in tests/heap_allocation.rs

#![no_std]
#![no_main]
#![feature(custom_test_frameworks)]
#![test_runner(blog_os::test_runner)]
#![reexport_test_harness_main = "test_main"]

extern crate alloc;

use bootloader::{entry_point, BootInfo};
use core::panic::PanicInfo;

entry_point!(main);

fn main(boot_info: &'static BootInfo) -> ! {
    unimplemented!();
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

ما از توابع `test_runner` و `test_panic_handler` از `lib.rs` دوباره استفاده می‌کنیم. از آن‌جایی که می‌خواهیم تخصیص‌ها را تست کنیم، کریت `alloc` را از طریق عبارت `extern crate alloc` فعال می‌کنیم. برای اطلاعات بیشتر در مورد تست، پست [_تست کردن_] را بررسی کنید.

[_تست کردن_]: @/edition-2/posts/04-testing/index.fa.md

پیاده‌سازی تابع اصلی به صورت زیر است:

```rust
// in tests/heap_allocation.rs

fn main(boot_info: &'static BootInfo) -> ! {
    use blog_os::allocator;
    use blog_os::memory::{self, BootInfoFrameAllocator};
    use x86_64::VirtAddr;

    blog_os::init();
    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };
    allocator::init_heap(&mut mapper, &mut frame_allocator)
        .expect("heap initialization failed");

    test_main();
    loop {}
}
```

این بسیار شبیه به تابع `kernel_main` در `main.rs` است، با این تفاوت که `println` را فراخوانی نمی‌کنیم، هیچ تخصیص نمونه‌ای را درج نمی‌کنیم و `test_main` را بدون قید و شرط فراخوانی می‌کنیم.

اکنون آماده اضافه کردن چند مورد آزمایشی هستیم. ابتدا، یک تست اضافه می‌کنیم که برخی تخصیص‌های ساده را با استفاده از [`Box`] انجام می‌دهد و مقادیر تخصیص‌یافته را بررسی می‌کند تا اطمینان حاصل شود که تخصیص‌های اولیه کار می‌کنند:

```rust
// in tests/heap_allocation.rs
use alloc::boxed::Box;

#[test_case]
fn simple_allocation() {
    let heap_value_1 = Box::new(41);
    let heap_value_2 = Box::new(13);
    assert_eq!(*heap_value_1, 41);
    assert_eq!(*heap_value_2, 13);
}
```

مهم‌تر از همه، این تست تأیید می‌کند که هیچ خطایی در تخصیص رخ نمی‌دهد.

در مرحله بعد، ما به طور مکرر یک وِکتور بزرگ می‌سازیم تا هم تخصیص‌های بزرگ و هم تخصیص‌های متعدد (به دلیل تخصیص مجدد) را تست کنیم:

```rust
// in tests/heap_allocation.rs

use alloc::vec::Vec;

#[test_case]
fn large_vec() {
    let n = 1000;
    let mut vec = Vec::new();
    for i in 0..n {
        vec.push(i);
    }
    assert_eq!(vec.iter().sum::<u64>(), (n - 1) * n / 2);
}
```

ما حاصل جمع را با مقایسه آن با فرمول [n-th partial sum] تأیید می‌کنیم. این به ما اطمینان می‌دهد که مقادیر تخصیص داده شده همه درست هستند.

[n-th partial sum]: https://en.wikipedia.org/wiki/1_%2B_2_%2B_3_%2B_4_%2B_%E2%8B%AF#Partial_sums

به عنوان تست سوم، ده هزار تخصیص پشت سر هم ایجاد می‌کنیم:

```rust
// in tests/heap_allocation.rs

use blog_os::allocator::HEAP_SIZE;

#[test_case]
fn many_boxes() {
    for i in 0..HEAP_SIZE {
        let x = Box::new(i);
        assert_eq!(*x, i);
    }
}
```

این تست تضمین می‌کند که تخصیص‌دهنده از حافظه آزاد شده برای تخصیص‌های بعدی مجدد استفاده می‌کند، زیرا در غیر این صورت حافظه آن تمام می‌شود. ممکن است این یک نیاز آشکار برای یک تخصیص‌دهنده به نظر برسد، اما طرح‌های تخصیص‌دهنده‌ای وجود دارند که این کار را انجام نمی‌دهند. یک نمونه طراحی تخصیص‌دهنده bump است که در پست بعدی توضیح داده خواهد شد.

بیایید تست ادغام جدید را اجرا کنیم:

```
> cargo test --test heap_allocation
[…]
Running 3 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
```

هر سه تست موفق شدند! همچنین می‌توانید `تست محموله` (بدون آرگومان `--test`) را برای اجرای تمام تست‌های واحد و ادغام فراخوانی کنید.

## خلاصه

این پست مقدمه‌ای بر حافظه پویا ارائه می‌دهد و توضیح می‌دهد که چرا و کجا به آن نیاز است. دیدیم که چگونه بررسی‌کننده قرض راست از آسیب پذیری‌های رایج جلوگیری می‌کند و یاد گرفتیم که چگونه API تخصیص راست کار می‌کند.

پس از ایجاد یک پیاده‌سازی حداقلی از رابط تخصیص‌دهنده راست با استفاده از یک تخصیص‌دهنده ساختگی، یک محدوده حافظه هیپِ مناسب برای هسته ایجاد کردیم. برای آن، یک محدوده آدرس مجازی برای هیپ تعریف کردیم و سپس با استفاده از `Mapper` و `FrameAllocator` از پست قبلی، تمام صفحات آن محدوده را به فریم‌های فیزیکی نگاشت کردیم.

در نهایت، یک وابستگی به کریت `linked_list_allocator` اضافه کردیم تا یک تخصیص‌دهنده مناسب به هسته اضافه کنیم. با این تخصیص‌دهنده، توانستیم از `Box` ،`Vec` و دیگر انواع تخصیص و مجموعه از کریت `alloc` استفاده کنیم.

## مرحله بعدی چیست؟

در حالی که ما قبلاً پشتیبانی از تخصیص هیپ را در این پست اضافه کرده‌ایم، بیشتر کار را به جعبه `linked_list_allocator` واگذار کردیم. پست بعدی به تفصیل نشان می‌دهد که چگونه می‌توان یک تخصیص‌دهنده را از ابتدا پیاده‌سازی کرد. چندین طرح تخصیص‌دهنده ممکن را ارائه می‌دهد، نحوه پیاده‌سازی نسخه‌های ساده آن‌ها را نشان می‌دهد و مزایا و معایب آن‌ها را بیان می‌کند.
