+++
title = "پیاده‌سازی صفحه‌بندی"
weight = 9
path = "paging-implementation"
date = 2019-03-14

[extra]
chapter = "Memory Management"
# Please update this when updating the translation
translation_based_on_commit = "aa389dae4b0b756a13ae3c330fa9bdf55d5f5ba"
# GitHub usernames of the people that translated this post
translators = ["hamidrezakp", "MHBahrampour"]
rtl = true
+++

این پست نشان می‌دهد که چگونه می‌توان پشتیبانی از صفحه‌بندی را به هسته خودمان اضافه کنیم. ابتدا تکنیک‌های مختلف را بررسی می‌کند تا فریم‌های جدول صفحه فیزیکی را در اختیار هسته قرار دهد و مزایا و معایب مربوط به آن‌ها را مورد بحث قرار می‌دهد. سپس یک تابع ترجمه آدرس و یک تابع را برای ایجاد یک نگاشت جدید پیاده‌سازی می‌کند.

<!-- more -->

این بلاگ بصورت آزاد روی [گیت‌هاب] توسعه داده شده است. اگر شما مشکل یا سوالی دارید، لطفاً آن‌جا یک ایشو باز کنید. شما همچنین می‌توانید [در زیر] این پست کامنت بگذارید. منبع کد کامل این پست را می‌توانید در بِرَنچ [`post-09`][post branch] پیدا کنید.

[گیت‌هاب]: https://github.com/phil-opp/blog_os
[در زیر]: #comments
<!-- fix for zola anchor checker (target is in template): <a id="comments"> -->
[post branch]: https://github.com/phil-opp/blog_os/tree/post-09

<!-- toc -->

## مقدمه

[پست قبلی] مقدمه‌ای بر مفهوم صفحه‌بندی ارائه داد. صفحه‌بندی را با مقایسه آن با تقسیم‌بندی یک گزینه بهتر نشان داد، نحوه عملکرد صفحه‌بندی و جدول‌های صفحه را توضیح داد و سپس طراحی جدول صفحه 4 سطحی «x86_64» را معرفی کرد. ما متوجه شدیم که بوت‌لودر قبلاً یک سلسله مراتب جدول صفحه را برای هسته ما تنظیم کرده است، به این معنی که هسته ما قبلاً روی آدرس‌های مجازی اجرا می‌شود. این امر ایمنی را بهبود می‌بخشد زیرا دسترسی‌های غیرقانونی به حافظه باعث ایجاد استثناهای خطای صفحه به جای تغییر حافظه فیزیکی دلخواه می‌شود.

[پست قبلی]: @/edition-2/posts/08-paging-introduction/index.md

آن پست با این مشکل به پایان رسید که ما [نمی‌توانیم از هسته خود به جداول صفحه دسترسی پیدا کنیم][end of previous post] زیرا در حافظه فیزیکی ذخیره شده‌اند و هسته ما از قبل روی آدرس‌های مجازی اجرا می‌شود. این پست در این مرحله ادامه می‌یابد و روش‌های مختلف برای دسترسی به فریم‌های جدول صفحه برای هسته را بررسی می‌کند. ما مزایا و معایب هر رویکرد را مورد بحث قرار خواهیم داد و سپس برای یک رویکرد برای هسته خود تصمیم خواهیم گرفت.

[end of previous post]: @/edition-2/posts/08-paging-introduction/index.md#accessing-the-page-tables

برای پیاده‌سازی این رویکرد، به پشتیبانی بوت‌لودر نیاز داریم، بنابراین ابتدا آن را پیکربندی می‌کنیم. پس از آن، تابعی را پیاده‌سازی می‌کنیم که سلسله‌مراتب جدول صفحه را برای ترجمه آدرس‌های مجازی به فیزیکی طی می‌کند. در نهایت، نحوه ایجاد نگاشت‌های جدید در جدول‌های صفحه و نحوه یافتن فریم‌های حافظه استفاده نشده برای ایجاد جدول‌های صفحه جدید را یاد می‌گیریم.

## دسترسی به جدول‌های صفحه

دسترسی به جدول‌های صفحه از هسته ما آن‌قدرها هم که به نظر می‌رسد آسان نیست. برای درک مشکل، اجازه دهید دوباره به سلسله‌مراتب جدول صفحه 4 سطحی پست قبل نگاهی بیندازیم:

![An example 4-level page hierarchy with each page table shown in physical memory](../paging-introduction/x86_64-page-table-translation.svg)

نکته مهم در اینجا این است که هر ورودی صفحه، آدرس _فیزیکی_ جدول بعدی را ذخیره می‌کند. این امر از نیاز به اجرای ترجمه برای این آدرس‌ها نیز جلوگیری می‌کند، که برای عملکرد بد است و به راحتی می‌تواند حلقه‌های ترجمه بی‌پایان ایجاد کند.

مشکل این است که نمی‌توانیم مستقیماً به آدرس‌های فیزیکیِ هسته دسترسی پیدا کنیم، زیرا هسته روی آدرس‌های مجازی نیز اجرا می‌شود. برای مثال، وقتی به آدرس «4KiB» دسترسی پیدا می‌کنیم، به آدرس _مجازی_ «4KiB» دسترسی پیدا می‌کنیم، نه به آدرس _فیزیکی_ «4KiB» جایی که جدول صفحه سطح 4 در آن ذخیره می‌شود. وقتی می‌خواهیم به آدرس فیزیکی «4KiB» دسترسی پیدا کنیم، فقط می‌توانیم از طریق برخی از آدرس‌های مجازی که به آن نگاشت می‌شوند، این کار را انجام دهیم.

بنابراین برای دسترسی به فریم‌های جدول صفحه، باید برخی از صفحات مجازی را به آن‌ها نگاشت کنیم. راه‌های مختلفی برای ایجاد این نگاشت‌ها وجود دارد که همگی به ما امکان دسترسی به فریم‌های جدول صفحه دلخواه را می‌دهند.

### نگاشت هویت

یک راه حل ساده این است که **تمام جداول صفحه را نگاشت هویت کرد**:

![A virtual and a physical address space with various virtual pages mapped to the physical frame with the same address](identity-mapped-page-tables.svg)

در این مثال، ما قاب‌های جدول صفحه با نگاشت هویت شده مختلف را می‌بینیم. به این ترتیب آدرس‌های فیزیکی جداول صفحه نیز آدرس‌های مجازی معتبری هستند تا بتوانیم به راحتی به جداول صفحه همه سطوح که اولین آن ثبات CR3 است دسترسی داشته باشیم.

با این حال، فضای آدرس مجازی را به هم ریخته و یافتن بخش منطقه‌های پیوسته با اندازه‌های بزرگتر را دشوارتر می‌کند. به عنوان مثال، تصور کنید که می‌خواهیم یک منطقه حافظه مجازی به اندازه 1000 کیلوبایت در گرافیک بالا ایجاد کنیم، برای مثال برای [نگاشت حافظه یک فایل]. ما نمی‌توانیم منطقه را با «28KiB» شروع کنیم، زیرا با صفحه از قبل نگاشت شده در «1004KiB» برخورد می‌کند. بنابراین باید بیشتر جستجو کنیم تا زمانی که یک منطقه به اندازه کافی بزرگ و بدون نگاشت پیدا کنیم، برای مثال در `1008KiB`. این یک مشکل تکه‌تکه شدن مشابه با [تقسیم‌بندی] است.

[نگاشت حافظه یک فایل]: https://en.wikipedia.org/wiki/Memory-mapped_file
[تقسیم‌بندی]: @/edition-2/posts/08-paging-introduction/index.md#fragmentation

به همین ترتیب، ایجاد جداول صفحه جدید را بسیار دشوارتر می‌کند، زیرا ما باید فریم‌های فیزیکی را پیدا کنیم که صفحات مربوطه آن‌ها قبلاً مورد استفاده قرار نگرفته‌اند. به عنوان مثال، فرض کنید که منطقه حافظه _مجازی_ 1000KiB را که از '1008KiB' شروع می‌شود برای فایل نگاشت حافظه خود رزرو کرده‌ایم. اکنون دیگر نمی‌توانیم از هیچ فریمی با آدرس _فیزیکی_ بین «1000KiB» و «2008KiB» استفاده کنیم، زیرا نمی‌توانیم آن را نگاشت کنیم.

### نگاشت در یک آفست ثابت

برای جلوگیری از به هم ریختگی فضای آدرس مجازی، می‌توانیم **از یک ناحیه حافظه جداگانه برای نگاشت جدول صفحه استفاده کنیم**. بنابراین به جای نگاشت هویت فریم های جدول صفحه، آن ها را با یک آفست ثابت در فضای آدرس مجازی نگاشت می‌کنیم. به عنوان مثال، آفست می تواند 10TiB باشد:

![The same figure as for the identity mapping, but each mapped virtual page is offset by 10 TiB.](page-tables-mapped-at-offset.svg)

با استفاده از حافظه مجازی در محدوده `10TiB..(10TiB + physical memory size)` به طور انحصاری برای نگاشت جدول صفحه، از مشکلات برخورد (collision) نگاشت هویت جلوگیری می‌کنیم. رزرو چنین منطقه بزرگی از فضای آدرس مجازی تنها در صورتی امکان‌پذیر است که فضای آدرس مجازی بسیار بزرگتر از اندازه حافظه فیزیکی باشد. این مشکل در x86_64 نیست زیرا فضای آدرس 48 بیتی دارای بزرگی 256TiB است.

این رویکرد هنوز هم این عیب را دارد که هر زمان که جدول صفحه جدیدی ایجاد می‌کنیم باید یک نگاشت جدید ایجاد کنیم. همچنین، اجازه دسترسی به جداول صفحه سایر فضاهای آدرس را نمی‌دهد، که در هنگام ایجاد یک فرآیند جدید مفید خواهد بود.

### نگاشت کردن حافظه فیزیکی به صورت کامل

ما می توانیم این مشکلات را با **نگاشت کردن حافظه فیزیکی به صورت کامل** به جای فقط فریم‌های جدول صفحه، حل کنیم:

![The same figure as for the offset mapping, but every physical frame has a mapping (at 10TiB + X) instead of only page table frames.](map-complete-physical-memory.svg)

این رویکرد به هسته ما اجازه می‌دهد تا به حافظه فیزیکی دلخواه، از جمله فریم‌های جدول صفحه سایر فضاهای آدرس دسترسی پیدا کند. محدوده حافظه مجازی رزرو شده همان اندازه قبلی است، با این تفاوت که دیگر شامل صفحات نگاشت نشده نیست.

نقطه ضعف این رویکرد این است که جداول صفحه اضافی برای ذخیره‌سازی نگاشت حافظه فیزیکی مورد نیاز است. این جداول صفحه باید در جایی ذخیره شوند، بنابراین بخشی از حافظه فیزیکی را مصرف می‌کنند، که می‌تواند در دستگاه‌هایی که حافظه کمی دارند مشکل ساز شود.

با این حال، در x86_64، به جای صفحات پیش‌فرض 4KiB، می‌توانیم از [صفحات عظیم] با اندازه 2MiB برای نگاشت کردن استفاده کنیم. به این ترتیب، نگاشت 32GiB حافظه فیزیکی تنها به 132KiB برای جداول صفحه نیاز دارد زیرا تنها به یک جدول سطح 3 و 32 جدول سطح 2 نیاز است. صفحات بزرگ نیز از آن‌جایی که از ورودی‌های کمتری در TLB استفاده می‌کنند، در حافظه نهان کارآمدتر هستند.

[صفحات عظیم]: https://en.wikipedia.org/wiki/Page_%28computer_memory%29#Multiple_page_sizes

### نگاشت موقت

برای دستگاه‌هایی که حافظه فیزیکی بسیار کمی دارند، می‌توانیم **قاب‌های جداول صفحه را فقط به طور موقت** در زمانی که نیاز به دسترسی داشته باشیم نگاشت کنیم. برای اینکه بتوانیم نگاشت‌های موقت را ایجاد کنیم، فقط به یک جدول سطح 1 با نگاشت هویت شده نیاز داریم:

![A virtual and a physical address space with an identity mapped level 1 table, which maps its 0th entry to the level 2 table frame, thereby mapping that frame to page with address 0](temporarily-mapped-page-tables.svg)

جدول سطح 1 در این تصویر 2MiB اول فضای آدرس مجازی را کنترل می‌کند. این به این خاطر که با شروع از ثبات CR3 و دنبال کردن ورودی 0 در جداول صفحه سطح 4، سطح 3 و سطح 2 قابل دسترسی است. ورودی با اندیس «8» صفحه مجازی را در آدرس «32KiB» به فریم فیزیکی در آدرس «32KiB» نگاشت می‌کند، بنابراین هویت خود جدول سطح 1 را نگاشت می‌کند. تصویر این نگاشت هویت را با پیکان افقی در `32KiB` نشان می‌دهد.

با نوشتن در جدول سطح 1 نگاشت هویت شده، هسته ما می‌تواند تا 511 نگاشت موقت (512 منهای ورودی مورد نیاز برای نگاشت هویت) ایجاد کند. در مثال بالا، هسته دو نگاشت موقت ایجاد کرد:

- با نگاشت ورودی 0 جدول سطح 1 به فریم با آدرس `24KiB`، یک نگاشت موقت از صفحه مجازی در `0KiB` به فریم فیزیکی جدول صفحه سطح 2 ایجاد کرد که با فلش خط‌چین شده مشخص است.
- با نگاشت نهمین ورودی جدول سطح 1 به فریم با آدرس '4KiB'، یک نگاشت موقت از صفحه مجازی در `36KiB` به فریم فیزیکی جدول صفحه سطح 4 ایجاد کرد که با فلش خط‌چین شده مشخص است.

اکنون هسته می‌تواند با نوشتن در صفحه '0KiB' به جدول صفحه سطح 2 و با نوشتن در صفحه '36KiB' به جدول صفحه سطح 4 دسترسی پیدا کند.

فرآیند دسترسی به یک قاب جدول صفحه دلخواه با نگاشت موقت به صورت زیر خواهد بود:

- یک ورودی آزاد را در جدول سطح 1 نگاشت هویت شده جستجو کنید.
- آن ورودی را به فریم فیزیکی جدول صفحه‌ای که می‌خواهیم به آن دسترسی داشته باشیم نگاشت کنید.
- از طریق صفحه مجازی که به ورودی نگاشت می‌دهد، به فریم هدف دسترسی پیدا کنید.
- ورودی را به حالت unused برگردانید و در نتیجه نگاشت موقت را دوباره حذف کنید.

این رویکرد از همان 512 صفحه مجازی برای ایجاد نگاشت‌ها استفاده مجدد می‌کند و بنابراین تنها به 4KiB حافظه فیزیکی نیاز دارد. اشکال این است که کمی دست و پا گیر می‌باشد، به خصوص از آن‌جایی که یک نگاشت جدید ممکن است به تغییراتی در سطوح مختلف جدول نیاز داشته باشد، به این معنی که ما باید چندین بار روند بالا را تکرار کنیم.

### جدول‌های صفحه بازگشتی

یکی دیگر از رویکردهای جالب، که به هیچ وجه به جداول صفحه اضافی نیاز ندارد، **نگاشت جدول صفحه به صورت بازگشتی** است. ایده پشت این رویکرد این است که برخی از ورودی‌های جدول صفحه سطح 4 را به خود جدول سطح 4 نگاشت کنیم. با انجام این کار، ما به طور موثر بخشی از فضای آدرس مجازی را رزرو می‌کنیم و تمام فریم‌های جدول صفحه فعلی و آینده را به آن فضا نگاشت می‌کنیم.

بیایید یک مثال را مرور کنیم تا بفهمیم همه این‌ها چگونه کار می‌کنند:

![An example 4-level page hierarchy with each page table shown in physical memory. Entry 511 of the level 4 page is mapped to frame 4KiB, the frame of the level 4 table itself.](recursive-page-table.png)

تنها تفاوت با [مثال موجود در ابتدای این پست] ورودی اضافی در اندیس «511» در جدول سطح 4 است که به فریم فیزیکی «4KiB» نگاشت شده است، همان قاب خود جدول سطح 4.

[مثال موجود در ابتدای این پست]: #accessing-page-tables

با اجازه دادن به CPU که این ورودی را در ترجمه دنبال کند، به جدول سطح 3 نمی‌رسد، بلکه دوباره به همان جدول سطح 4 می‌رسد. این شبیه به یک تابع بازگشتی است که خودش را فراخوانی می‌کند، بنابراین این جدول یک جدول صفحه بازگشتی نامیده می‌شود. نکته مهم این است که CPU فرض می‌کند که هر ورودی در جدول سطح 4 به جدول سطح 3 اشاره می‌کند، بنابراین اکنون جدول سطح 4 را به عنوان جدول سطح 3 در نظر می‌گیرد. این مورد به این دلیل کار می‌کند که جداول همه سطوح دقیقاً طرح یکسانی در x86_64 دارند.

با دنبال کردن ورودی بازگشتی یک یا چند بار قبل از شروع ترجمه واقعی، می‌توانیم تعداد سطوحی را که CPU طی می‌کند، به طور مؤثر کوتاه کنیم. به عنوان مثال، اگر یک بار ورودی بازگشتی را دنبال کنیم و سپس به جدول سطح 3 برویم، CPU فکر می‌کند که جدول سطح 3 یک جدول سطح 2 است. در ادامه، جدول سطح 2 را به عنوان جدول سطح 1 و جدول سطح 1 را به عنوان فریم نگاشت شده در نظر می‌گیرد. این بدان معنی است که اکنون می‌توانیم خواندن و نوشتن را برای جدول صفحه سطح 1 انجام دهیم زیرا CPU فکر می‌کند که فریم نگاشت شده است. تصویر زیر 5 مرحله ترجمه را نشان می‌دهد:

![The above example 4-level page hierarchy with 5 arrows: "Step 0" from CR4 to level 4 table, "Step 1" from level 4 table to level 4 table, "Step 2" from level 4 table to level 3 table, "Step 3" from level 3 table to level 2 table, and "Step 4" from level 2 table to level 1 table.](recursive-page-table-access-level-1.png)

به همین ترتیب، می‌توانیم قبل از شروع ترجمه، ورودی بازگشتی را دو بار دنبال کنیم تا تعداد سطوح پیموده شده را به دو کاهش دهیم:

![The same 4-level page hierarchy with the following 4 arrows: "Step 0" from CR4 to level 4 table, "Steps 1&2" from level 4 table to level 4 table, "Step 3" from level 4 table to level 3 table, and "Step 4" from level 3 table to level 2 table.](recursive-page-table-access-level-2.png)

بیایید مرحله به مرحله آن را مرور کنیم: ابتدا CPU ورودی بازگشتی جدول سطح 4 را دنبال می‌کند و فکر می‌کند که به جدول سطح 3 می‌رسد. سپس دوباره ورودی بازگشتی را دنبال می‌کند و فکر می‌کند که به جدول سطح 2 می‌رسد. اما در واقعیت همچنان در جدول سطح 4 قرار دارد. وقتی CPU اکنون ورودی دیگری را دنبال می‌کند، در جدول سطح 3 قرار می‌گیرد اما فکر می‌کند که از قبل در جدول سطح 1 قرار دارد. بنابراین در حالی که ورودی بعدی به جدول سطح 2 اشاره می‌کند، CPU فکر می‌کند که به فریم نگاشت شده اشاره می‌کند که این امکان را به ما می‌دهد که خواندن و نوشتن را برای جدول سطح 2 انجام دهیم.


دسترسی به جداول سطوح 3 و 4 نیز به همین صورت انجام شدنی است. برای دسترسی به جدول سطح 3، ورودی بازگشتی را سه بار دنبال می‌کنیم، و CPU را فریب می‌دهیم تا فکر کند در جدول سطح 1 قرار دارد. سپس ورودی دیگری را دنبال می‌کنیم و به جدول سطح 3 می‌رسیم که CPU آن را به عنوان یک فریم نگاشت شده در نظر می‌گیرد. برای دسترسی به خود جدول سطح 4، فقط ورودی بازگشتی را چهار بار دنبال می‌کنیم تا زمانی که CPU خود جدول سطح 4 را به عنوان قاب نگاشت شده (به رنگ آبی در تصویر زیر) در نظر بگیرد.

![The same 4-level page hierarchy with the following 3 arrows: "Step 0" from CR4 to level 4 table, "Steps 1,2,3" from level 4 table to level 4 table, and "Step 4" from level 4 table to level 3 table. In blue the alternative "Steps 1,2,3,4" arrow from level 4 table to level 4 table.](recursive-page-table-access-level-3.png)

ممکن است کمی طول بکشد تا این مفهوم را به خوبی درک کنید، اما در عمل کاملاً خوب عمل می‌کند.

در بخش زیر نحوه ساخت آدرس‌های مجازی برای دنبال کردن ورودی بازگشتی یک یا چند بار را توضیح می‌دهیم. برای پیاده‌سازی خود از صفحه‌بندی بازگشتی استفاده نخواهیم کرد، بنابراین برای ادامه پست نیازی به خواندن آن نیست. اگر به علاقه‌مند هستید، روی _"Address Calculation"_ کلیک کنید و به خواندن ادامه دهید.

---

<details>
<summary><h4>Address Calculation</h4></summary>

دیدیم که می‌توانیم با دنبال کردن ورودی بازگشتی یک یا چند بار قبل از ترجمه واقعی به جداول همه سطوح دسترسی داشته باشیم. از آن‌جایی که اندیس‌های جداول چهار سطح مستقیماً از آدرس مجازی مشتق شده‌اند، باید آدرس‌های مجازی خاصی برای این تکنیک بسازیم. به یاد داشته باشید، فهرست‌های جدول صفحه از آدرس به روش زیر مشتق می‌شوند:

![Bits 0–12 are the page offset, bits 12–21 the level 1 index, bits 21–30 the level 2 index, bits 30–39 the level 3 index, and bits 39–48 the level 4 index](../paging-introduction/x86_64-table-indices-from-address.svg)

بیایید فرض کنیم که می خواهیم به جدول صفحه سطح 1 دسترسی داشته باشیم که یک صفحه خاص را نگاشت می‌کند. همان‌طور که در بالا یاد گرفتیم، این بدان معنی است که قبل از ادامه با اندیس‌های سطح 4، سطح 3 و سطح 2 باید یک بار ورودی بازگشتی را دنبال کنیم. برای انجام این کار، هر بلوک آدرس را یک بلوک به سمت راست منتقل می‌کنیم و اندیس سطح 4 اصلی را به اندیس ورودی بازگشتی تنظیم می‌کنیم:

![Bits 0–12 are the offset into the level 1 table frame, bits 12–21 the level 2 index, bits 21–30 the level 3 index, bits 30–39 the level 4 index, and bits 39–48 the index of the recursive entry](table-indices-from-address-recursive-level-1.svg)

برای دسترسی به جدول سطح 2 آن صفحه، هر بلوکِ اندیس را دو بلوک به سمت راست منتقل می‌کنیم و هر دو بلوک اندیس سطح 4 اصلی و اندیس سطح 3 اصلی را به اندیس ورودی بازگشتی تنظیم می‌کنیم:

![Bits 0–12 are the offset into the level 2 table frame, bits 12–21 the level 3 index, bits 21–30 the level 4 index, and bits 30–39 and bits 39–48 are the index of the recursive entry](table-indices-from-address-recursive-level-2.svg)

دسترسی به جدول سطح 3، با جابجا کردن هر بلوک، به اندازه سه بلوک به سمت راست و استفاده از اندیس بازگشتی برای بلوک‌های آدرس اصلی سطح 4، سطح 3 و سطح 2 کار می‌کند:

![Bits 0–12 are the offset into the level 3 table frame, bits 12–21 the level 4 index, and bits 21–30, bits 30–39 and bits 39–48 are the index of the recursive entry](table-indices-from-address-recursive-level-3.svg)

در نهایت، می‌توانیم با جابجایی هر بلوک به اندازه چهار بلوک به سمت راست و استفاده از اندیس بازگشتی برای همه بلوک‌های آدرس به جز برای آفست، به جدول سطح 4 دسترسی پیدا کنیم:

![Bits 0–12 are the offset into the level l table frame and bits 12–21, bits 21–30, bits 30–39 and bits 39–48 are the index of the recursive entry](table-indices-from-address-recursive-level-4.svg)

اکنون می‌توانیم آدرس‌های مجازی را برای جداول صفحه هر چهار سطح محاسبه کنیم. ما حتی می‌توانیم آدرسی را محاسبه کنیم که دقیقاً به یک ورودی جدول صفحه خاص اشاره می‌کند، با ضرب اندیس آن در 8، همان اندازه ورودی جدول صفحه.

جدول زیر ساختار آدرس برای دسترسی به انواع مختلف فریم‌ها را خلاصه می‌کند:

Virtual Address for | Address Structure ([octal])
------------------- | -------------------------------
Page                | `0o_SSSSSS_AAA_BBB_CCC_DDD_EEEE`
Level 1 Table Entry | `0o_SSSSSS_RRR_AAA_BBB_CCC_DDDD`
Level 2 Table Entry | `0o_SSSSSS_RRR_RRR_AAA_BBB_CCCC`
Level 3 Table Entry | `0o_SSSSSS_RRR_RRR_RRR_AAA_BBBB`
Level 4 Table Entry | `0o_SSSSSS_RRR_RRR_RRR_RRR_AAAA`

[octal]: https://en.wikipedia.org/wiki/Octal

در حالی که «AAA» شاخص سطح 4، «BBB» شاخص سطح 3، «CCC» شاخص سطح 2، و «DDD» شاخص سطح 1 قاب نگاشت شده، و «EEEE» آفست در آن است. "RRR" شاخص ورودی بازگشتی است. هنگامی که یک شاخص (سه رقم) به یک آفست (چهار رقم) تبدیل می‌شود، با ضرب آن در 8 (اندازه یک ورودی جدول صفحه) انجام می‌شود. با این آفست، آدرس حاصل مستقیماً به ورودی جدول صفحه مربوطه اشاره می‌کند.

رشته `SSSSSS` بیت‌های پسوند علامت هستند، به این معنی که همه آن‌ها کپی بیت 47 هستند. این یک الزام ویژه برای آدرس‌های معتبر در معماری x86_64 است. در [پست قبلی][sign extension] توضیح دادیم.

[sign extension]: @/edition-2/posts/08-paging-introduction/index.md#paging-on-x86-64

ما از اعداد [octal] برای نشان دادن آدرس‌ها استفاده می‌کنیم زیرا هر کاراکتر اکتال نشان دهنده سه بیت است که به ما اجازه می‌دهد به وضوح نمایه‌های 9 بیتی سطوح مختلف جدول صفحه را از هم جدا کنیم. این کار با سیستم هگزادسیمال که هر کاراکتر نشان دهنده چهار بیت است امکان پذیر نیست.

##### In Rust Code

برای ساخت چنین آدرس‌هایی به زبان راست، می‌توانید از عملیات بیتی استفاده کنید:

```rust
// the virtual address whose corresponding page tables you want to access
let addr: usize = […];

let r = 0o777; // recursive index
let sign = 0o177777 << 48; // sign extension

// retrieve the page table indices of the address that we want to translate
let l4_idx = (addr >> 39) & 0o777; // level 4 index
let l3_idx = (addr >> 30) & 0o777; // level 3 index
let l2_idx = (addr >> 21) & 0o777; // level 2 index
let l1_idx = (addr >> 12) & 0o777; // level 1 index
let page_offset = addr & 0o7777;

// calculate the table addresses
let level_4_table_addr =
    sign | (r << 39) | (r << 30) | (r << 21) | (r << 12);
let level_3_table_addr =
    sign | (r << 39) | (r << 30) | (r << 21) | (l4_idx << 12);
let level_2_table_addr =
    sign | (r << 39) | (r << 30) | (l4_idx << 21) | (l3_idx << 12);
let level_1_table_addr =
    sign | (r << 39) | (l4_idx << 30) | (l3_idx << 21) | (l2_idx << 12);
```

کد بالا فرض می‌کند که آخرین ورودی سطح 4 با شاخص `0o777` (511) به صورت بازگشتی نگاشت شده است. در حال حاضر این‌طور نیست، بنابراین کد هنوز کار نخواهد کرد. در زیر ببینید که چگونه به بوت‌لودر بگوییم نگاشت بازگشتی را تنظیم کند.

به‌جای انجام عملیات بیتی با دست، می‌توانید از نوع ['RecursivePageTable'] جعبه 'x86_64' استفاده کنید، که انتزاعات ایمن را برای عملیات‌های مختلف جدول صفحه فراهم می‌کند. به عنوان مثال، کد زیر نحوه ترجمه یک آدرس مجازی را به آدرس فیزیکی نگاشت شده آن نشان می‌دهد:

[`RecursivePageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.RecursivePageTable.html

```rust
// in src/memory.rs

use x86_64::structures::paging::{Mapper, Page, PageTable, RecursivePageTable};
use x86_64::{VirtAddr, PhysAddr};

/// Creates a RecursivePageTable instance from the level 4 address.
let level_4_table_addr = […];
let level_4_table_ptr = level_4_table_addr as *mut PageTable;
let recursive_page_table = unsafe {
    let level_4_table = &mut *level_4_table_ptr;
    RecursivePageTable::new(level_4_table).unwrap();
}


/// Retrieve the physical address for the given virtual address
let addr: u64 = […]
let addr = VirtAddr::new(addr);
let page: Page = Page::containing_address(addr);

// perform the translation
let frame = recursive_page_table.translate_page(page);
frame.map(|frame| frame.start_address() + u64::from(addr.page_offset()))
```

باز هم، یک نگاشت بازگشتی معتبر برای این کد مورد نیاز است. با چنین نگاشتی، «level_4_table_addr» گم شده را می‌توان مانند مثال کد اول محاسبه کرد.
Again, a valid recursive mapping is required for this code. With such a mapping, the missing `level_4_table_addr` can be calculated as in the first code example.

</details>

---

صفحه‌بندی بازگشتی یک تکنیک جالب است که نشان می‌دهد یک نگاشت واحد در یک جدول صفحه چقدر می‌تواند قدرتمند باشد. پیاده‌سازی آن نسبتاً آسان است و فقط به حداقل مقدار راه‌اندازی نیاز دارد (فقط یک ورودی بازگشتی)، بنابراین انتخاب خوبی برای اولین آزمایش‌های صفحه‌بندی است.

با این حال، معایبی نیز دارد:

- حجم زیادی از حافظه مجازی (512 گیگابایت) را اشغال می‌کند. این یک مشکل بزرگ در فضای آدرس بزرگ 48 بیتی نیست، اما ممکن است منجر به کاهش بهینه بودن رفتار حافظه نهان شود.
- فقط اجازه می‌دهد تا به راحتی به فضای آدرس فعال فعلی دسترسی پیدا کنید. دسترسی به سایر فضاهای آدرس همچنان با تغییر ورودی بازگشتی امکان‌پذیر است، اما یک نقشه موقت برای جابجایی مجدد لازم است. نحوه انجام این کار را در پست [_Remap The Kernel_] (منسوخ شده) توضیح دادیم.
- به شدت به فرمت جدول صفحه x86 متکی است و ممکن است روی معماری‌های دیگر کار نکند.

[_Remap The Kernel_]: https://os.phil-opp.com/remap-the-kernel/#overview

## پشتیبانی از بوت‌لودر

همه این رویکردها برای تنظیم خود نیاز به اصلاحات جدول صفحه دارند. به عنوان مثال، نقشه برای حافظه فیزیکی باید ایجاد شود یا ورودی جدول سطح 4 به صورت بازگشتی نگاشت شود. مشکل این است که ما نمی‌توانیم این نگاشت‌های مورد نیاز را بدون یک راه موجود برای دسترسی به جداول صفحه ایجاد کنیم.

این بدان معناست که ما به کمک بوت‌لودر نیاز داریم که جداول صفحه‌ای را که هسته ما روی آن اجرا می‌شود ایجاد می‌کند. بوت‌لودر به جداول صفحه دسترسی دارد، بنابراین می‌تواند هر نگاشتی را که ما نیاز داریم ایجاد کند. در پیاده‌سازی فعلی، جعبه «bootloader» از دو رویکرد فوق پشتیبانی می‌کند که از طریق [ویژگی‌های کارگو] کنترل می‌شوند:

[cargo features]: https://doc.rust-lang.org/cargo/reference/features.html#the-features-section

- ویژگی 'map_physical_memory' حافظه فیزیکی را به صورت کامل در فضای آدرس مجازی نگاشت می‌کند. بنابراین، هسته می‌تواند به تمام حافظه فیزیکی دسترسی داشته باشد و می‌تواند از رویکرد [_Map the Complete Physical Memory_](#map-the-complete-physical-memory) پیروی کند.
- با ویژگی `recursive_page_table`،  بوت‌لودر ورودی جدول صفحه سطح 4 را به صورت بازگشتی ترسیم می‌کند. این به هسته اجازه می‌دهد تا به جداول صفحه همان‌طور که در بخش [_Recursive Page Tables_](#recursive-page-tables) توضیح داده شده است، دسترسی پیدا کند.

ما اولین رویکرد را برای هسته خود انتخاب می‌کنیم زیرا ساده، مستقل از پلتفرم و قدرتمندتر است (همچنین امکان دسترسی به قاب‌های که مربوط به جدول صفحه نیستند را نیز فراهم می‌کند). برای فعال کردن پشتیبانی بوت‌لودر مورد نیاز، ویژگی «map_physical_memory» را به وابستگی «bootloader» خود اضافه می‌کنیم:

```toml
[dependencies]
bootloader = { version = "0.9.8", features = ["map_physical_memory"]}
```

با فعال بودن این ویژگی، بوت لودر حافظه فیزیکی را به صورت کامل به محدوده آدرس مجازی استفاده نشده نگاشت می‌کند. برای برقراری ارتباط محدوده آدرس مجازی با هسته ما، بوت‌لودر ساختار _boot information_ را ارسال می‌کند.

### اطلاعات بوت

جعبه «bootloader» یک ساختمان ['BootInfo'] را تعریف می‌کند که حاوی تمام اطلاعاتی است که به هسته ما ارسال می‌کند. این ساختمان هنوز در مرحله اولیه است، بنابراین هنگام بروزرسانی به نسخه‌های بوت‌لودر [semver-incompatible] آینده، انتظار کمی مشکل را داشته باشید. با فعال بودن ویژگی `map_physical_memory`، در حال حاضر دارای دو فیلد `memory_map` و `physical_memory_offset` است:

[`BootInfo`]: https://docs.rs/bootloader/0.9.3/bootloader/bootinfo/struct.BootInfo.html
[semver-incompatible]: https://doc.rust-lang.org/stable/cargo/reference/specifying-dependencies.html#caret-requirements

- قسمت `memory_map` شامل نمای کلی حافظه فیزیکی موجود است. این به هسته ما می‌گوید که چه مقدار حافظه فیزیکی در سیستم موجود است و کدام مناطق حافظه برای دستگاه‌هایی مانند سخت افزار VGA رزرو شده است. نقشه حافظه را می‌توان از ثابت‌افزار (firmware) BIOS یا UEFI جستجو کرد، اما فقط در اوایل فرآیند بوت. به همین دلیل، باید توسط بوت‌لودر ارائه شود، زیرا هیچ راهی برای بازیابی آن توسط هسته وجود ندارد. در ادامه این پست به نقشه حافظه نیاز خواهیم داشت.
- مقدار `physical_memory_offset` آدرس شروع مجازی نگاشت حافظه فیزیکی را به ما می‌گوید. با افزودن این آفست به یک آدرس فیزیکی، آدرس مجازی مربوطه را دریافت می‌کنیم. این به ما اجازه می‌دهد تا به حافظه فیزیکی دلخواه از هسته خود دسترسی پیدا کنیم.

بوت‌لودر ساختمان `BootInfo` را به شکل آرگومان `&'static BootInfo` به تابع `_start` به هسته ارسال می‌کند. ما هنوز این آرگومان را در تابع خود نداریم، پس بیایید آن را اضافه کنیم:

```rust
// in src/main.rs

use bootloader::BootInfo;

#[no_mangle]
pub extern "C" fn _start(boot_info: &'static BootInfo) -> ! { // new argument
    […]
}
```

رها کردن این آرگومان قبلاً مشکلی نبود زیرا قرارداد فراخوانی x86_64 اولین آرگومان را در یک ثبات CPU ارسال می‌کند. بنابراین، زمانی که آرگومان اعلام نشده باشد به سادگی نادیده گرفته می‌شود. با این حال، اگر به طور تصادفی از یک نوع آرگومان اشتباه استفاده کنیم، مشکل ایجاد می‌شود، زیرا کامپایلر امضای نوع صحیح تابع نقطه ورودی ما را نمی‌داند.

### ماکروی `entry_point`

از آن‌جایی که تابع '_start' ما به صورت خارجی از بوت‌لودر فراخوانی می‌شود، هیچ بررسی‌ای از امضای تابع ما انجام نمی‌شود. این بدان معناست که می‌توانیم اجازه دهیم آرگومان‌های دلخواه دریافت کند بدون این‌که هیچ خطای کامپایلی رخ دهد، اما در زمان اجرا شکست می‌خورد یا باعث رفتار نامشخص می‌شود.

برای اطمینان از این‌که تابعِ نقطه ورودی همیشه دارای امضای صحیحی است که بوت‌لودر انتظار دارد، جعبه «bootloader» یک ماکرو [«entry_point»] ارائه می‌کند که یک روش بررسی شده برای تعریف تابع راست به عنوان نقطه ورودی ارائه می‌کند. بیایید تابع نقطه ورودی خود را برای استفاده از این ماکرو بازنویسی کنیم:

[`entry_point`]: https://docs.rs/bootloader/0.6.4/bootloader/macro.entry_point.html

```rust
// in src/main.rs

use bootloader::{BootInfo, entry_point};

entry_point!(kernel_main);

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
}
```

دیگر نیازی به استفاده از `extern "C"` یا `no_mangle` برای نقطه ورودی خود نداریم، زیرا ماکرو نقطه ورودی سطح پایین‌تر `_start` را برای ما تعریف می‌کند. تابع «kernel_main» اکنون یک تابع راست کاملاً عادی است، بنابراین می‌توانیم یک نام دلخواه برای آن انتخاب کنیم. نکته مهم این است که نوع بررسی شود تا زمانی که از یک امضای تابع اشتباه استفاده می‌کنیم، مثلاً با اضافه کردن یک آرگومان یا تغییر نوع آرگومان، خطای کامپایل رخ می‌دهد.

بیایید همان تغییر را در `lib.rs` نیز انجام دهیم:

```rust
// in src/lib.rs

#[cfg(test)]
use bootloader::{entry_point, BootInfo};

#[cfg(test)]
entry_point!(test_kernel_main);

/// Entry point for `cargo test`
#[cfg(test)]
fn test_kernel_main(_boot_info: &'static BootInfo) -> ! {
    // like before
    init();
    test_main();
    hlt_loop();
}
```

از آن‌جایی که نقطه ورودی فقط در حالت تست استفاده می‌شود، ویژگی `#[cfg(test)]` را به همه موارد اضافه می‌کنیم. ما به نقطه ورودی آزمایشی خود نام متمایز «test_kernel_main» می‌دهیم تا از اشتباه گرفتن «kernel_main» از «main.rs» جلوگیری کنیم. ما فعلاً از پارامتر «BootInfo» استفاده نمی‌کنیم، بنابراین نام پارامتر را با یک پیشوند «_» می‌گذاریم تا هشدار متغیر استفاده نشده خاموش شود.

## پیاده‌سازی

اکنون که به حافظه فیزیکی دسترسی داریم، در نهایت می‌توانیم شروع به پیاده‌سازی کد جدول صفحه خود کنیم. ابتدا نگاهی به جداول صفحه‌ای که در حال حاضر فعال هستند که هسته ما روی آن‌ها اجرا می‌شود خواهیم انداخت. در مرحله دوم، یک تابع ترجمه ایجاد می‌کنیم که آدرس فیزیکی را که آدرس مجازی داده شده به آن نگاشت شده است، برمی‌گرداند. به عنوان آخرین مرحله، سعی می‌کنیم جداول صفحه را به منظور ایجاد یک نگاشت جدید تغییر دهیم.

قبل از شروع، ما یک ماژول `memory` جدید برای کد خود ایجاد می‌کنیم:

```rust
// in src/lib.rs

pub mod memory;
```

برای ماژول یک فایل خالی `src/memory.rs` ایجاد می‌کنیم.

### دسترسی به جدول صفحه‌ها

در [پایان پست قبلی]، سعی کردیم به جداول صفحه‌ای که هسته ما روی آن‌ها اجرا می‌شود نگاهی بیندازیم، اما موفق نشدیم زیرا نتوانستیم به فریم فیزیکی که ثبات `CR3` به آن اشاره می‌کند دسترسی پیدا کنیم. اکنون می‌توانیم با ایجاد یک تابع «active_level_4_table» که مرجعی را به جدول صفحه فعال سطح 4 برمی‌گرداند، از ادامه پست قبل کار را از سر بگیریم:

[پایان پست قبلی]: @/edition-2/posts/08-paging-introduction/index.md#accessing-the-page-tables

```rust
// in src/memory.rs

use x86_64::{
    structures::paging::PageTable,
    VirtAddr,
};

/// Returns a mutable reference to the active level 4 table.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`. Also, this function must be only called once
/// to avoid aliasing `&mut` references (which is undefined behavior).
pub unsafe fn active_level_4_table(physical_memory_offset: VirtAddr)
    -> &'static mut PageTable
{
    use x86_64::registers::control::Cr3;

    let (level_4_table_frame, _) = Cr3::read();

    let phys = level_4_table_frame.start_address();
    let virt = physical_memory_offset + phys.as_u64();
    let page_table_ptr: *mut PageTable = virt.as_mut_ptr();

    &mut *page_table_ptr // unsafe
}
```

ابتدا قاب فیزیکی جدول فعال سطح 4 را از ثبات `CR3` می‌خوانیم. سپس آدرس شروع فیزیکی آن را می‌گیریم، آن را به «u64» تبدیل می‌کنیم، و آن را به «physical_memory_offset» اضافه می‌کنیم تا آدرس مجازی که در آن قاب جدول صفحه نگاشت شده است را به دست آوریم. در نهایت، آدرس مجازی را از طریق متد «as_mut_ptr» به یک اشاره‌گر خام «*mut PageTable» تبدیل می‌کنیم و سپس یک مرجع «&mut PageTable» از آن ایجاد می‌کنیم. یک مرجع «&mut» به جای مرجع «&» ایجاد می‌کنیم زیرا در ادامه این پست، جداول صفحه را تغییر می‌دهیم.

ما در اینجا نیازی به استفاده از یک بلوک ناامن نداریم زیرا راست با کل بدنه یک «unsafe fn» مانند یک بلوک بزرگ «unsafe» رفتار می‌کند. این باعث می‌شود کد ما خطرناک‌تر شود، زیرا می‌توانیم به طور تصادفی یک عملیات ناامن را بدون توجه به خطوط قبلی تعریف کنیم. همچنین تشخیص عملیات ناامن را بسیار دشوارتر می‌کند. یک [RFC](https://github.com/rust-lang/rfcs/pull/2585) برای تغییر این رفتار وجود دارد.

اکنون می‌توانیم از این تابع برای چاپ ورودی‌های جدول سطح 4 استفاده کنیم:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory::active_level_4_table;
    use x86_64::VirtAddr;

    println!("Hello World{}", "!");
    blog_os::init();

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let l4_table = unsafe { active_level_4_table(phys_mem_offset) };

    for (i, entry) in l4_table.iter().enumerate() {
        if !entry.is_unused() {
            println!("L4 Entry {}: {:?}", i, entry);
        }
    }

    // as before
    #[cfg(test)]
    test_main();

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

ابتدا، `physical_memory_offset` از ساختمان `BootInfo` را به یک [`VirtAddr`] تبدیل می‌کنیم و آن را به تابع `active_level_4_table` عبور می‌دهیم. سپس از تابع `iter` برای پیمایش روی ورودی‌های جدول صفحه و از ترکیب‌کننده [`Enumerate`] برای اضافه کردن یک شاخص `i` به هر عنصر استفاده می‌کنیم. ما فقط ورودی‌های غیر خالی را چاپ می‌کنیم زیرا همه 512 ورودی روی صفحه جا نمی‌شوند.

[`VirtAddr`]: https://docs.rs/x86_64/0.14.2/x86_64/addr/struct.VirtAddr.html
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate

وقتی آن را اجرا می‌کنیم، خروجی زیر را می‌بینیم:

![QEMU printing entry 0 (0x2000, PRESENT, WRITABLE, ACCESSED), entry 1 (0x894000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 31 (0x88e000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 175 (0x891000, PRESENT, WRITABLE, ACCESSED, DIRTY), and entry 504 (0x897000, PRESENT, WRITABLE, ACCESSED, DIRTY)](qemu-print-level-4-table.png)

می بینیم که ورودی‌های غیر خالی مختلفی وجود دارد که همگی به جداول سطح 3 مختلفی نگاشت می‌شوند. ناحیه‌های بسیار زیادی وجود دارد زیرا کد هسته، پشته هسته، نقشه حافظه فیزیکی و اطلاعات بوت همگی از ناحیه‌های حافظه جداگانه استفاده می‌کنند.

برای پیمایش بیشتر جداول صفحه و نگاهی به جدول سطح 3، می‌توانیم فریم نگاشت شده یک ورودی را گرفته و دوباره آن را به یک آدرس مجازی تبدیل کنیم:

```rust
// in the `for` loop in src/main.rs

use x86_64::structures::paging::PageTable;

if !entry.is_unused() {
    println!("L4 Entry {}: {:?}", i, entry);

    // get the physical address from the entry and convert it
    let phys = entry.frame().unwrap().start_address();
    let virt = phys.as_u64() + boot_info.physical_memory_offset;
    let ptr = VirtAddr::new(virt).as_mut_ptr();
    let l3_table: &PageTable = unsafe { &*ptr };

    // print non-empty entries of the level 3 table
    for (i, entry) in l3_table.iter().enumerate() {
        if !entry.is_unused() {
            println!("  L3 Entry {}: {:?}", i, entry);
        }
    }
}
```

برای مشاهده جداول سطح 2 و سطح 1، این روند را برای ورودی‌های سطح 3 و سطح 2 تکرار می‌کنیم. همان‌طور که می‌توانید تصور کنید، این به سرعت بسیار شلوغ و طولانی می‌شود، بنابراین ما کد کامل را در اینجا نشان نمی‌دهیم.

پیمایش جداول صفحه به صورت دستی جالب است زیرا به درک نحوه انجام ترجمه توسط CPU کمک می‌کند. با این حال، بیشتر اوقات ما فقط به آدرس فیزیکی نگاشت شده برای یک آدرس مجازی معین را می‌خواهیم، بنابراین بیایید یک تابع برای آن ایجاد کنیم.

### ترجمه آدرس‌ها

برای ترجمه یک آدرس مجازی به فیزیکی، باید جدول چهار سطحی صفحه را طی کنیم تا به فریم نگاشت شده برسیم. بیایید تابعی ایجاد کنیم که این ترجمه را انجام دهد:

```rust
// in src/memory.rs

use x86_64::PhysAddr;

/// Translates the given virtual address to the mapped physical address, or
/// `None` if the address is not mapped.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`.
pub unsafe fn translate_addr(addr: VirtAddr, physical_memory_offset: VirtAddr)
    -> Option<PhysAddr>
{
    translate_addr_inner(addr, physical_memory_offset)
}
```

ما تابع را به یک تابع `translate_addr_inner` ایمن ارسال می‌کنیم تا دامنه `unsafe` را محدود کنیم. همان‌طور که در بالا اشاره کردیم، راست با کل بدنه یک fn ناامن مانند یک بلوک ناامن بزرگ رفتار می‌کند. با فراخوانی یک تابع امن خصوصی، هر عملیات «unsafe» را دوباره آشکار می‌کنیم.

تابع داخلی خصوصی شامل پیاده‌سازی واقعی است:

```rust
// in src/memory.rs

/// Private function that is called by `translate_addr`.
///
/// This function is safe to limit the scope of `unsafe` because Rust treats
/// the whole body of unsafe functions as an unsafe block. This function must
/// only be reachable through `unsafe fn` from outside of this module.
fn translate_addr_inner(addr: VirtAddr, physical_memory_offset: VirtAddr)
    -> Option<PhysAddr>
{
    use x86_64::structures::paging::page_table::FrameError;
    use x86_64::registers::control::Cr3;

    // read the active level 4 frame from the CR3 register
    let (level_4_table_frame, _) = Cr3::read();

    let table_indexes = [
        addr.p4_index(), addr.p3_index(), addr.p2_index(), addr.p1_index()
    ];
    let mut frame = level_4_table_frame;

    // traverse the multi-level page table
    for &index in &table_indexes {
        // convert the frame into a page table reference
        let virt = physical_memory_offset + frame.start_address().as_u64();
        let table_ptr: *const PageTable = virt.as_ptr();
        let table = unsafe {&*table_ptr};

        // read the page table entry and update `frame`
        let entry = &table[index];
        frame = match entry.frame() {
            Ok(frame) => frame,
            Err(FrameError::FrameNotPresent) => return None,
            Err(FrameError::HugeFrame) => panic!("huge pages not supported"),
        };
    }

    // calculate the physical address by adding the page offset
    Some(frame.start_address() + u64::from(addr.page_offset()))
}
```

به جای استفاده مجدد از تابع `active_level_4_table`، دوباره فریم سطح 4 را از ثبات `CR3` می‌خوانیم. این کار را انجام می‌دهیم زیرا پیاده‌سازی این نمونه اولیه را ساده می‌کند. نگران نباشید، بزودی یک راه حل بهتری ایجاد خواهیم کرد.

ساختمان «VirtAddr» متدهایی را برای محاسبه شاخص‌ها در جدول‌های صفحه چهار سطح ارائه می‌دهد. ما این شاخص‌ها را در یک آرایه کوچک ذخیره می‌کنیم زیرا به ما اجازه می‌دهد تا با استفاده از حلقه «for» از جداول صفحه را پیمایش کنیم. خارج از حلقه، آخرین «frame» بازدید شده را به خاطر می‌آوریم تا بعداً آدرس فیزیکی را محاسبه کنیم. «frame» در حین پیمایش به فریم‌های جدول صفحه و بعد از آخرین پیمایش، یعنی پس از دنبال کردن ورودی سطح 1، به قاب نگاشت شده اشاره می‌کند.

در داخل حلقه، ما دوباره از «physical_memory_offset» برای تبدیل فریم به مرجع جدول صفحه استفاده می‌کنیم. سپس ورودی جدول صفحه فعلی را می‌خوانیم و از تابع [`PageTableEntry::frame`] برای بازیابی فریم نگاشت شده استفاده می‌کنیم. اگر ورودی به یک فریم نگاشت نشده باشد، `None` را برمی‌گردانیم. اگر ورودی یک صفحه عظیم 2MiB یا 1GiB را نگاشت کند، فعلاً پنیک می‌کنیم.

[`PageTableEntry::frame`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/page_table/struct.PageTableEntry.html#method.frame

بیایید تابع ترجمه خود را با ترجمه چند آدرس آزمایش کنیم:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // new import
    use blog_os::memory::translate_addr;

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);

    let addresses = [
        // the identity-mapped vga buffer page
        0xb8000,
        // some code page
        0x201008,
        // some stack page
        0x0100_0020_1a10,
        // virtual address mapped to physical address 0
        boot_info.physical_memory_offset,
    ];

    for &address in &addresses {
        let virt = VirtAddr::new(address);
        let phys = unsafe { translate_addr(virt, phys_mem_offset) };
        println!("{:?} -> {:?}", virt, phys);
    }

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

وقتی آن را اجرا می‌کنیم، خروجی زیر را می‌بینیم:

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, "panicked at 'huge pages not supported'](qemu-translate-addr.png)

همان‌طور که انتظار می‌رود، آدرس نگاشت هویتی «0xb8000» به همان آدرس فیزیکی ترجمه می‌شود. صفحه کد و صفحه پشته به برخی آدرس‌های فیزیکی دلخواه ترجمه می‌شوند، که بستگی به نحوه ایجاد نگاشت اولیه توسط بوت‌لودر برای هسته ما دارد. شایان ذکر است که 12 بیت آخر همیشه پس از ترجمه ثابت می‌مانند، که منطقی است زیرا این بیت‌ها [_page offset_] هستند و بخشی از ترجمه نمی‌باشند.

[_page offset_]: @/edition-2/posts/08-paging-introduction/index.md#paging-on-x86-64

از آن‌جایی که با افزودن «offset_physical_memory_offset» می‌توان به هر آدرس فیزیکی دسترسی داشت، ترجمه آدرس «physical_memory_offset» باید به آدرس فیزیکی «0» اشاره کند. با این حال، ترجمه با شکست مواجه می‌شود زیرا نقشه از صفحات بزرگ برای کارایی استفاده می‌کند، که هنوز در پیاده‌سازی ما پشتیبانی نمی‌شود.

### استفاده از `OffsetPageTable`

ترجمه آدرس‌های مجازی به فیزیکی یک کار رایج در هسته سیستم‌عامل است، بنابراین جعبه «x86_64» یک انتزاع برای آن فراهم می‌کند. این پیاده‌سازی در حال حاضر از صفحات بزرگ و چندین توابع دیگر جدول صفحه به غیر از «translate_addr» پشتیبانی می‌کند، بنابراین به‌جای افزودن پشتیبانی صفحه بزرگ به پیاده‌سازی خود، از آن در ادامه استفاده خواهیم کرد.

پایه این انتزاع دو صفت هستند که توابع مختلف نقشه جدول صفحه را تعریف می‌کنند:

- صفت ['Mapper'] در اندازه صفحه عمومی است و توابعی را ارائه می‌کند که کارهایی را روی صفحات انجام می‌دهند. مثال‌ها عبارتند از [`translate_page`]، که یک صفحه معین را به فریمی با همان اندازه ترجمه می‌کند، و [`map_to`]، که نگاشت جدیدی در جدول صفحه ایجاد می‌کند.
- صفت [`Translate`] توابعی را ارائه می‌دهد که با اندازه‌های مختلف از صفحه کار می‌کنند، برای مثال [`translate_addr`] یا [`translate`] عمومی .

[`Mapper`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html
[`translate_page`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html#tymethod.translate_page
[`map_to`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html#method.map_to
[`Translate`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html
[`translate_addr`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html#method.translate_addr
[`translate`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html#tymethod.translate

صفت‌ها فقط رابط (interface) را تعریف می‌کنند، آن‌ها هیچ پیاده‌سازی را ارائه نمی‌دهند. جعبه `x86_64` در حال حاضر سه نوع را ارائه می‌دهد که صفات را با الزامات مختلف اجرا می‌کند. نوع ['OffsetPageTable'] فرض می‌کند که حافظه فیزیکی به صورت کامل با مقداری افست به فضای آدرس مجازی نگاشت شده است. ['MappedPageTable'] کمی انعطاف پذیرتر است: فقط نیاز دارد که هر قاب جدول صفحه به فضای آدرس مجازی در یک آدرس قابل محاسبه نگاشت شود. در نهایت، از نوع ['RecursivePageTable'] می‌توان برای دسترسی به فریم‌های جدول صفحه از طریق [جدول صفحه بازگشتی](#recursive-page-tables) استفاده کرد.

[`OffsetPageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.OffsetPageTable.html
[`MappedPageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MappedPageTable.html
[`جدول صفحه بازگشتی`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.RecursivePageTable.html

در مورد ما، بوت‌لودر حافظه فیزیکی را به صورت کامل در یک آدرس مجازی مشخص شده توسط متغیر `physical_memory_offset` نگاشت می‌کند، بنابراین می‌توانیم از نوع `OffsetPageTable` استفاده کنیم. برای مقداردهی اولیه آن، یک تابع «init» جدید در ماژول «memory» ایجاد می‌کنیم:

```rust
use x86_64::structures::paging::OffsetPageTable;

/// Initialize a new OffsetPageTable.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`. Also, this function must be only called once
/// to avoid aliasing `&mut` references (which is undefined behavior).
pub unsafe fn init(physical_memory_offset: VirtAddr) -> OffsetPageTable<'static> {
    let level_4_table = active_level_4_table(physical_memory_offset);
    OffsetPageTable::new(level_4_table, physical_memory_offset)
}

// make private
unsafe fn active_level_4_table(physical_memory_offset: VirtAddr)
    -> &'static mut PageTable
{…}
```

این تابع «physical_memory_offset» را به عنوان آرگومان می‌گیرد و یک نمونه «OffsetPageTable» جدید را با طول عمر «'static» برمی‌گرداند. یعنی نمونه برای کل زمان اجرای هسته ما معتبر می‌ماند. در بدنه تابع، ابتدا تابع 'active_level_4_table' را فراخوانی می‌کنیم تا یک مرجع قابل تغییر به جدول صفحه سطح 4 بازیابی شود. سپس تابع ['OffsetPageTable::new'] را با این مرجع فراخوانی می‌کنیم. به عنوان پارامتر دوم، تابع `new` توقع یک آدرس مجازی را دارد که در آن نگاشت حافظه فیزیکی آغاز می‌شود، این آدرس مجازی در متغیر `physical_memory_offset` داده شده است.

[`OffsetPageTable::new`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.OffsetPageTable.html#method.new

تابع 'active_level_4_table' از این پس باید فقط از تابع 'init' فراخوانی شود زیرا در صورتی که چندین بار فراخوانی شود به راحتی می‌تواند به مراجع تغییرپذیر مستعار (aliased mutable references) منجر شود که می‌تواند باعث رفتار نامشخص شود. به همین دلیل، با حذف مشخص کننده «pub» تابع را خصوصی می‌کنیم.

اکنون می‌توانیم به جای تابع «memory::translate_addr» از روش «Translate::translate_addr» استفاده کنیم. ما فقط باید چند خط را در «kernel_main» تغییر دهیم:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // new: different imports
    use blog_os::memory;
    use x86_64::{structures::paging::Translate, VirtAddr};

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    // new: initialize a mapper
    let mapper = unsafe { memory::init(phys_mem_offset) };

    let addresses = […]; // same as before

    for &address in &addresses {
        let virt = VirtAddr::new(address);
        // new: use the `mapper.translate_addr` method
        let phys = mapper.translate_addr(virt);
        println!("{:?} -> {:?}", virt, phys);
    }

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

برای استفاده از روش [`translate_addr`]، باید صفت `Translate` که این متد را ارائه می‌کند، به کد خود وارد کنیم.

وقتی اکنون آن را اجرا می‌کنیم، همان نتایج ترجمه قبلی را می‌بینیم، با این تفاوت که حالا ترجمه صفحه عظیم نیز کار می‌کند:

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, 0x18000000000 -> 0x0](qemu-mapper-translate-addr.png)

همان‌طور که انتظار می‌رفت، ترجمه‌های «0xb8000» و آدرس‌های کد و پشته مانند تابع ترجمه خودمان باقی می‌مانند. علاوه بر این، اکنون می‌بینیم که آدرس مجازی «physical_memory_offset» به آدرس فیزیکی «0x0» نگاشت شده است.

با استفاده از تابع ترجمه از نوع «MappedPageTable» می‌توانیم از زحمت پیاده‌سازی پشتیبانی صفحه بزرگ صرف نظر کنیم. همچنین به سایر توابع صفحه مانند 'map_to' دسترسی داریم که در بخش بعدی از آن‌ها استفاده خواهیم کرد.

در این مرحله ما دیگر نیازی به توابع `memory::translate_addr` و `memory::translate_addr_inner` نداریم، بنابراین می‌توانیم آن‌ها را حذف کنیم.

### ایجاد کردن یک نگاشت جدید

تا به حال ما فقط به جداول صفحه نگاه می‌کردیم و چیزی را تغییر نمی‌دادیم. بیایید تغییر رویه داده و یک نقشه جدید برای صفحه‌ای که از قبل نقشه نشده، ایجاد کنیم.

ما از تابع ['map_to'] صفت ['Mapper'] برای پیاده‌سازی خود استفاده خواهیم کرد، بنابراین اجازه دهید ابتدا به آن تابع نگاهی بیندازیم. مستندات به ما می‌گویند که چهار آرگومان نیاز دارد: صفحه‌ای که می‌خواهیم نگاشت کنیم، فریمی که صفحه باید به آن نگاشت شود، مجموعه‌ای از پرچم‌ها برای ورودی جدول صفحه، و یک «frame_allocator». تخصیص‌دهنده فریم را نیاز داریم زیرا نگاشت صفحه داده شده ممکن است نیاز به ایجاد جداول صفحه اضافی داشته باشد که به فریم‌های استفاده نشده به عنوان حافظه پشتیبان نیاز دارند.

[`map_to`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.Mapper.html#tymethod.map_to
[`Mapper`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.Mapper.html

#### یک تابع `create_example_mapping`

اولین مرحله از پیاده‌سازی ایجاد یک تابع «create_example_mapping» جدید است که یک صفحه مجازی داده شده را به «0xb8000»، همان قاب فیزیکی بافر متن VGA نگاشت می‌کند. ما آن فریم را انتخاب می‌کنیم زیرا به ما اجازه می‌دهد به راحتی تست کنیم که آیا نگاشت به درستی ایجاد شده است یا خیر: فقط باید در صفحه جدید نقشه شده بنویسیم و ببینیم که نوشته روی صفحه ظاهر می‌شود یا خیر.

تابع «create_example_mapping» به شکل زیر است:

```rust
// in src/memory.rs

use x86_64::{
    PhysAddr,
    structures::paging::{Page, PhysFrame, Mapper, Size4KiB, FrameAllocator}
};

/// Creates an example mapping for the given page to frame `0xb8000`.
pub fn create_example_mapping(
    page: Page,
    mapper: &mut OffsetPageTable,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) {
    use x86_64::structures::paging::PageTableFlags as Flags;

    let frame = PhysFrame::containing_address(PhysAddr::new(0xb8000));
    let flags = Flags::PRESENT | Flags::WRITABLE;

    let map_to_result = unsafe {
        // FIXME: this is not safe, we do it only for testing
        mapper.map_to(page, frame, flags, frame_allocator)
    };
    map_to_result.expect("map_to failed").flush();
}
```

علاوه بر `page` که باید نگاشت شود، این تابع انتظار یک مرجع قابل تغییر به یک نمونه `OffsetPageTable` و یک `frame_allocator` را دارد. پارامتر «frame_allocator» از نحو [`impl Trait`][impl-trait-arg] استفاده می‌کند تا در همه انواعی که صفت [`FrameAllocator`] را پیاده‌سازی می‌کنند، [عمومی] (generic) باشد. این صفت نسبت به صفت ['Size'] عمومی است تا هم با صفحات استاندارد 4KiB و هم با صفحات عظیم 2MiB/1GiB کار کند. ما فقط می‌خواهیم یک نقشه 4KiB ایجاد کنیم، بنابراین پارامتر عمومی را روی `Size4KiB` تنظیم می کنیم.

[impl-trait-arg]: https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters
[عمومی]: https://doc.rust-lang.org/book/ch10-00-generics.html
[`FrameAllocator`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.FrameAllocator.html
[`PageSize`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/page/trait.PageSize.html

روش ['map_to'] ناامن است زیرا فراخواننده باید اطمینان حاصل کند که فریم قبلاً استفاده نشده است. زیرا دوبار نگاشت یک فریم می‌تواند منجر به رفتار نامشخص شود، برای مثال زمانی که دو مرجع مختلف «&mut» به مکان حافظه فیزیکی یکسانی اشاره می‌کنند. در مورد ما، از فریم بافر متن VGA، که قبلاً نقشه شده است، مجدداً استفاده می‌کنیم، بنابراین شرط لازم را می‌شکنیم. با این حال، تابع `create_example_mapping` تنها یک تابع آزمایشی موقت است و پس از این پست حذف خواهد شد، بنابراین مشکلی ندارد. برای یادآوری ناامنی، یک کامنت «FIXME» اضافه می‌کنیم.

علاوه بر «page» و «unused_frame»، روش «map_to» مجموعه‌ای از پرچم‌ها را برای نگاشت و ارجاع به «frame_allocator» می‌گیرد که بزودی توضیح داده خواهد شد. برای پرچم‌ها، ما پرچم «PRESENT» را تنظیم می‌کنیم، زیرا برای همه ورودی‌های معتبر می‌باشد و پرچم «WRITABLE» برای قابلیت نوشتن صفحه نقشه شده لازم است. برای فهرست همه پرچم‌های ممکن، بخش [_Page Table Format_] پست قبلی را ببینید.

[_Page Table Format_]: @/edition-2/posts/08-paging-introduction/index.md#page-table-format

تابع [`map_to`] ممکن است شکست بخورد، بنابراین یک [`Result`] را برمی‌گرداند. از آن‌جایی که این فقط چند نمونه کد است که نیازی به خوب و استاندارد بودن ندارد، ما فقط از ['expect'] استفاده می‌کنیم تا در هنگام بروز خطا پنیک کنیم. در صورت موفقیت، این تابع یک نوع ['MapperFlush'] را برمی‌گرداند که راه آسانی را برای پاک کردن صفحه جدید نقشه شده از TLB با روش ['flush'] خود ارائه می‌دهد. مانند «Result»، این نوع از صفت [`#[must_use]`][must_use] استفاده می‌کند تا زمانی که اشتباهاً استفاده از آن را فراموش می‌کنیم، هشداری صادر کند.

[`Result`]: https://doc.rust-lang.org/core/result/enum.Result.html
[`expect`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.expect
[`MapperFlush`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MapperFlush.html
[`flush`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MapperFlush.html#method.flush
[must_use]: https://doc.rust-lang.org/std/result/#results-must-be-used

#### یک `FrameAllocator` ساختگی

برای اینکه بتوانیم «create_example_mapping» را فراخوانی کنیم، باید نوعی ایجاد کنیم که ابتدا صفت «FrameAllocator» را پیاده‌سازی کند. همان‌طور که در بالا ذکر شد، این صفت در صورتی که «map_to» نیاز داشته باشد، مسئول تخصیص فریم‌ها برای جدول صفحه جدید است.

بیایید با حالت ساده شروع کنیم و فرض کنیم که نیازی به ایجاد جداول صفحه جدید نداریم. برای این مورد، یک تخصیص‌دهنده فریم که همیشه «None» را برمی‌گرداند، کافی است. ما یک «EmptyFrameAllocator» را برای تست تابع نقشه خود ایجاد می‌کنیم:

```rust
// in src/memory.rs

/// A FrameAllocator that always returns `None`.
pub struct EmptyFrameAllocator;

unsafe impl FrameAllocator<Size4KiB> for EmptyFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> {
        None
    }
}
```

پیاده‌سازی «FrameAllocator» ناامن است زیرا پیاده‌ساز باید تضمین کند که تخصیص‌دهنده فقط فریم‌های استفاده‌نشده را ارائه می‌دهد. در غیر این صورت ممکن است رفتار نامشخصی رخ دهد، برای مثال زمانی که دو صفحه مجازی در یک قاب فیزیکی نگاشت می‌شوند. «EmptyFrameAllocator» ما فقط «None» را برمی‌گرداند، بنابراین در این مورد مشکلی نیست.

#### انتخاب یک صفحه مجازی

ما اکنون یک تخصیص‌دهنده فریم ساده داریم که می‌توانیم آن را به تابع `create_example_mapping` خود منتقل کنیم. با این حال، تخصیص‌دهنده همیشه «None» را برمی‌گرداند، بنابراین این کار تنها در صورتی جواب می‌دهد که برای ایجاد نقشه به فریم‌های جدول صفحه اضافی نیاز نباشد. برای درک این‌که چه زمانی به فریم‌های جدول صفحه اضافی نیاز است و چه زمانی نیاز نیست، بیایید مثالی را در نظر بگیریم:

![A virtual and a physical address space with a single mapped page and the page tables of all four levels](required-page-frames-example.svg)

تصویر بالا فضای آدرس مجازی را در سمت چپ، فضای آدرس فیزیکی در سمت راست و جداول صفحه را در بین آن‌ها نشان می‌دهد. جداول صفحه در فریم‌های حافظه فیزیکی ذخیره می‌شوند که با خط‌چین نشان داده می‌شوند. فضای آدرس مجازی حاوی یک صفحه نگاشت شده در آدرس «0x803fe00000» است که با رنگ آبی مشخص شده است. برای ترجمه این صفحه به فریمش، CPU جدول صفحه 4 سطحی را تا زمانی که به فریم آدرس 36 کیلوبایت برسد، می‌پیماید.

علاوه بر این، تصویر بالا، قاب فیزیکی بافر متن VGA را به رنگ قرمز نشان می‌دهد. هدف ما این است که با استفاده از تابع `create_example_mapping` یک صفحه مجازی که قبلاً نقشه نشده بود را به این قاب نگاشت کنیم. از آن‌جایی که «EmptyFrameAllocator» ما همیشه «None» را برمی‌گرداند، می‌خواهیم نگاشت را طوری ایجاد کنیم که هیچ فریم اضافی از تخصیص‌دهنده مورد نیاز نباشد. این بستگی به صفحه مجازی‌ای دارد که برای نقشه انتخاب می‌کنیم.

این تصویر دو صفحه کاندید را در فضای آدرس مجازی نشان می‌دهد که هر دو با رنگ زرد مشخص شده‌اند. یک صفحه در آدرس «0x803fdfd000» است که 3 صفحه قبل از صفحه نقشه شده (به رنگ آبی) است. در حالی که شاخص‌های جدول صفحه سطح 4 و سطح 3 مانند صفحه آبی هستند، شاخص‌های سطح 2 و سطح 1 متفاوت هستند (به [پست قبلی][page-table-indices] مراجعه کنید). شاخص متفاوت در جدول سطح 2 به این معنی است که جدول سطح 1 متفاوتی برای این صفحه استفاده می‌شود. از آن‌جایی که این جدول سطح 1 هنوز وجود ندارد، اگر آن صفحه را برای نگاشت مثال خود انتخاب کنیم، باید ابتدا آن را ایجاد کنیم، که به یک فریم فیزیکی استفاده نشده اضافی نیاز دارد. در مقابل، دومین صفحه نامزد در آدرس «0x803fe02000» این مشکل را ندارد زیرا از همان جدول صفحه سطح 1 نسبت به صفحه آبی استفاده می‌کند. بنابراین، تمام جداول صفحه مورد نیاز از قبل وجود دارد.

[page-table-indices]: @/edition-2/posts/08-paging-introduction/index.md#paging-on-x86-64

به طور خلاصه، دشواری ایجاد یک نقشه جدید به صفحه مجازی که می خواهیم نقشه کنیم بستگی دارد. در ساده‌ترین حالت، جدول صفحه سطح 1 برای صفحه از قبل وجود دارد و ما فقط باید یک ورودی بنویسیم. در سخت‌ترین حالت، صفحه در یک منطقه حافظه است که هنوز سطح 3 وجود ندارد، بنابراین ابتدا باید جداول صفحه سطح 3، سطح 2 و سطح 1 جدید ایجاد کنیم.

برای فراخوانی تابع «create_example_mapping» با «EmptyFrameAllocator»، باید صفحه‌ای را انتخاب کنیم که همه جداول صفحه از قبل وجود داشته باشد. برای یافتن چنین صفحه‌ای، می‌توانیم از این واقعیت استفاده کنیم که بوت‌لودر، خود را در اولین مگابایت فضای آدرس مجازی بارگذاری می‌کند. یعنی یک جدول سطح 1 معتبر برای همه صفحات این منطقه وجود دارد. بنابراین، می‌توانیم هر صفحه بدون استفاده در این منطقه حافظه را برای نقشه مثال خود انتخاب کنیم، مانند صفحه موجود در آدرس «0». به طور معمول، این صفحه باید بدون استفاده بماند تا تضمین کند که از بین بردن ارجاع یک اشاره‌گر تهی باعث خطای صفحه می‌شود، بنابراین می‌دانیم که بوت‌لودر آن را بدون نقشه رها می‌کند.

#### ایجاد کردن نگاشت

ما اکنون تمام پارامترهای لازم برای فراخوانی تابع `create_example_mapping` خود را داریم، بنابراین بیایید تابع `kernel_main` را تغییر دهیم تا صفحه را در آدرس مجازی `0` ترسیم کنیم. از آن‌جایی که صفحه را به قاب بافر متن VGA نگاشت می‌کنیم، باید بتوانیم بعد از آن روی صفحه بنویسیم. پیاده‌سازی به شکل زیر است:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory;
    use x86_64::{structures::paging::Page, VirtAddr}; // new import

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = memory::EmptyFrameAllocator;

    // map an unused page
    let page = Page::containing_address(VirtAddr::new(0));
    memory::create_example_mapping(page, &mut mapper, &mut frame_allocator);

    // write the string `New!` to the screen through the new mapping
    let page_ptr: *mut u64 = page.start_address().as_mut_ptr();
    unsafe { page_ptr.offset(400).write_volatile(0x_f021_f077_f065_f04e)};

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

ابتدا با فراخوانی تابع `create_example_mapping` با یک مرجع قابل تغییر به نمونه‌های `mapper` و `frame_allocator` نگاشت صفحه را در آدرس `0` ایجاد می‌کنیم. این صفحه را به قاب بافر متنی VGA نگاشت می‌کند، بنابراین باید هر گونه نوشتن روی آن را روی صفحه ببینیم.

سپس صفحه را به یک اشاره‌گر خام تبدیل می‌کنیم و مقداری را در آفست `400` می‌نویسیم. در ابتدای صفحه نمی‌نویسیم زیرا خط بالای بافر VGA مستقیماً توسط `println` بعدی از صفحه خارج می‌شود. مقدار `0x_f021_f077_f065_f04e` را می‌نویسیم که نشان‌دهنده رشته _"New!"_ در پس‌زمینه سفید است. همان‌طور که [در پست _"حالت متن VGA"_] آموختیم، نوشته‌های بافر VGA باید فرار (volatile) باشد، بنابراین از روش [`write_volatile`] استفاده می‌کنیم.

[در پست _"حالت متن VGA"_]: @/edition-2/posts/03-vga-text-buffer/index.md#volatile
[`write_volatile`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write_volatile

وقتی آن را در QEMU اجرا می‌کنیم، خروجی زیر را می‌بینیم:

![QEMU printing "It did not crash!" with four completely white cells in the middle of the screen](qemu-new-mapping.png)

رشته _"New!"_ که روی صفحه نشان داده شده است توسط نوشتن ما در صفحه `0` وجود دارد، به این معنی که ما با موفقیت یک نقشه جدید در جداول صفحه ایجاد کردیم.

ایجاد این نگاشت فقط به این دلیل کار می‌کند که جدول سطح 1 که مسئول صفحه در آدرس «0» می‌باشد، از قبل وجود دارد. وقتی می‌خواهیم صفحه‌ای را برای آن جدول سطح 1 ترسیم کنیم، تابع «map_to» با شکست مواجه می‌شود زیرا سعی می‌کند فریم‌هایی را از «EmptyFrameAllocator» برای ایجاد جداول صفحه جدید اختصاص دهد. وقتی می‌خواهیم صفحه «0xdeadbeaf000» را به جای «0» ترسیم کنیم، می‌توانیم این اتفاق را ببینیم:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
    let page = Page::containing_address(VirtAddr::new(0xdeadbeaf000));
    […]
}
```

وقتی آن را اجرا می‌کنیم، یک پنیک با پیام خطای زیر رخ می‌دهد:

```
panicked at 'map_to failed: FrameAllocationFailed', /…/result.rs:999:5
```

برای نگاشت صفحاتی که هنوز جدول صفحه سطح 1 ندارند، باید یک «FrameAllocator» مناسب ایجاد کنیم. اما چگونه بفهمیم کدام فریم‌ها استفاده نشده‌اند و چه مقدار حافظه فیزیکی در دسترس است؟

### تخصیص دادن فریم‌ها

برای ایجاد جداول صفحه جدید، باید یک تخصیص‌دهنده فریم مناسب ایجاد کنیم. برای این کار از «memory_map» استفاده می‌کنیم که توسط بوت‌لودر به عنوان بخشی از ساختمان «BootInfo» ارسال می‌شود:

```rust
// in src/memory.rs

use bootloader::bootinfo::MemoryMap;

/// A FrameAllocator that returns usable frames from the bootloader's memory map.
pub struct BootInfoFrameAllocator {
    memory_map: &'static MemoryMap,
    next: usize,
}

impl BootInfoFrameAllocator {
    /// Create a FrameAllocator from the passed memory map.
    ///
    /// This function is unsafe because the caller must guarantee that the passed
    /// memory map is valid. The main requirement is that all frames that are marked
    /// as `USABLE` in it are really unused.
    pub unsafe fn init(memory_map: &'static MemoryMap) -> Self {
        BootInfoFrameAllocator {
            memory_map,
            next: 0,
        }
    }
}
```

ساختمان دارای دو فیلد است: یک ارجاع `'static` به نقشه حافظه عبور داده شده توسط بوت‌لودر و یک فیلد `next` که تعداد فریم بعدی که تخصیص‌دهنده باید برگرداند را پیگیری کند.

همان‌طور که در بخش [_اطلاعات بوت_](#boot-information) توضیح دادیم، نقشه حافظه توسط ثابت‌افزار BIOS/UEFI ارائه شده است. فقط در مراحل اولیه بوت می‌توان آن را پرس‌وجو کرد، بنابراین بوت‌لودر از قبل توابع مربوطه را برای ما فراخوانی می‌کند. نقشه حافظه شامل فهرستی از ساختمان‌های ['MemoryRegion'] است که شامل آدرس شروع، طول و نوع (به عنوان مثال استفاده نشده، رزرو شده و غیره) هر ناحیه حافظه است.

تابع 'init' یک 'BootInfoFrameAllocator' را با یک نقشه حافظه داده شده مقداردهی اولیه می‌کند. فیلد «next» با «0» مقداردهی اولیه می‌شود و برای هر تخصیص فریم افزایش می‌یابد تا از دوبار بازگشت همان فریم جلوگیری شود. از آن‌جایی که نمی‌دانیم آیا فریم‌های قابل استفاده نقشه حافظه قبلاً در جای دیگری استفاده شده‌اند، تابع «init» ما باید «unsafe» باشد تا به تضمین‌های اضافی از فراخواننده نیاز داشته باشد.

#### یک متد `usable_frames`

قبل از اینکه صفت «FrameAllocator» را پیاده‌سازی کنیم، یک روش کمکی اضافه می‌کنیم که نقشه حافظه را به یک iterator از فریم‌های قابل استفاده تبدیل می‌کند:

```rust
// in src/memory.rs

use bootloader::bootinfo::MemoryRegionType;

impl BootInfoFrameAllocator {
    /// Returns an iterator over the usable frames specified in the memory map.
    fn usable_frames(&self) -> impl Iterator<Item = PhysFrame> {
        // get usable regions from memory map
        let regions = self.memory_map.iter();
        let usable_regions = regions
            .filter(|r| r.region_type == MemoryRegionType::Usable);
        // map each region to its address range
        let addr_ranges = usable_regions
            .map(|r| r.range.start_addr()..r.range.end_addr());
        // transform to an iterator of frame start addresses
        let frame_addresses = addr_ranges.flat_map(|r| r.step_by(4096));
        // create `PhysFrame` types from the start addresses
        frame_addresses.map(|addr| PhysFrame::containing_address(PhysAddr::new(addr)))
    }
}
```

این تابع از روش‌های ترکیب‌کننده iterator برای تبدیل «MemoryMap» اولیه به یک iterator از فریم‌های فیزیکی قابل استفاده، استفاده می‌کند:

- ابتدا متد «iter» را فراخوانی می‌کنیم تا نقشه حافظه را به یک iterator از [«MemoryRegion»]ها تبدیل کنیم.
- سپس از روش ['filter'] برای رد شدن (skip) از هر منطقه رزرو شده یا غیرقابل دسترس استفاده می‌کنیم. بوت‌لودر نقشه حافظه را برای تمام نگاشت‌هایی که ایجاد می‌کند بروز می‌کند، بنابراین فریم‌هایی که توسط هسته ما (کد، داده یا پشته) یا برای ذخیره اطلاعات بوت استفاده می‌شوند، قبلاً به عنوان `InUse` یا چیزی مشابه آن علامت‌گذاری شده‌اند. بنابراین می‌توانیم مطمئن باشیم که فریم‌های «Usable» در جای دیگری استفاده نمی‌شوند.
- پس از آن، از ترکیب‌کننده ['map'] و [نحو range] راست را استفاده می‌کنیم تا iterator مناطق حافظه خود را به یک iterator محدوده آدرس تبدیل کنیم.
- در مرحله بعد، از [`flat_map`] برای تبدیل محدوده آدرس به یک iterator آدرس‌های شروع فریم استفاده می‌کنیم و هر 4096مین آدرس را با استفاده از [`step_by`] انتخاب می‌کنیم. از آن‌جایی که 4096 بایت (= 4 کیلوبایت) اندازه صفحه است، آدرس شروع هر فریم را دریافت می‌کنیم. صفحه بوت‌لودر تمام ناحیه‌های حافظه قابل استفاده را تراز می‌کند تا در اینجا به هیچ تراز یا کد گرد کردنی (rounding code) نیاز نداشته باشیم. با استفاده از [`flat_map`] به جای «map»، یک `Iterator<Item = u64>` به جای `Iterator<Item = Iterator<Item = u64>>` دریافت می‌کنیم.
- در نهایت، آدرس‌های شروع را به انواع `PhysFrame` تبدیل می‌کنیم تا یک `Iterator<Item = PhysFrame>` بسازیم.

[`MemoryRegion`]: https://docs.rs/bootloader/0.6.4/bootloader/bootinfo/struct.MemoryRegion.html
[`filter`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.filter
[`map`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.map
[نحو range]: https://doc.rust-lang.org/core/ops/struct.Range.html
[`step_by`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.step_by
[`flat_map`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.flat_map

نوع برگشتی تابع از ویژگی [`impl Trait`] استفاده می‌کند. به این ترتیب، می‌توانیم تعیین کنیم که نوع خاصی را که صفت ['Iterator'] را با نوع آیتم `PhysFrame` پیاده‌سازی می‌کند، برگردانیم، اما نیازی به نام‌گذاری نوع بازگشتی مشخص نیست. این در اینجا مهم است زیرا ما _نمی‌توانیم_ نوع دقیق (concrete type) را نام ببریم چون به انواع closure غیرقابل نام‌گذاری (unnamable) بستگی دارد.

[`impl Trait`]: https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits
[`Iterator`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html

#### Implementing the `FrameAllocator` Trait

اکنون می‌توانیم ویژگی «FrameAllocator» را پیاده‌سازی کنیم:
Now we can implement the `FrameAllocator` trait:

```rust
// in src/memory.rs

unsafe impl FrameAllocator<Size4KiB> for BootInfoFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> {
        let frame = self.usable_frames().nth(self.next);
        self.next += 1;
        frame
    }
}
```

ابتدا از روش «قابی_مصرف» استفاده می کنیم تا یک تکرار کننده از فریم های قابل استفاده از نقشه حافظه به دست آوریم. سپس، از تابع [`Iterator::nth`] استفاده می کنیم تا فریم را با نمایه «self.next» بدست آوریم (در نتیجه از فریم های «(self.next - 1)» پرش می کنیم. قبل از برگرداندن آن فریم، `self.next` را یک عدد افزایش می دهیم تا در تماس بعدی فریم زیر را برگردانیم.
We first use the `usable_frames` method to get an iterator of usable frames from the memory map. Then, we use the [`Iterator::nth`] function to get the frame with index `self.next` (thereby skipping `(self.next - 1)` frames). Before returning that frame, we increase `self.next` by one so that we return the following frame on the next call.

[`Iterator::nth`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.nth

این پیاده‌سازی کاملاً بهینه نیست زیرا تخصیص‌دهنده «قابی_استفاده» را در هر تخصیص دوباره ایجاد می‌کند. بهتر است به‌جای آن، تکرارکننده را به‌عنوان یک فیلد ساختاری ذخیره کنید. سپس ما به روش «nth» نیازی نداریم و فقط می‌توانیم [«next»] را در هر تخصیص فراخوانی کنیم. مشکل این رویکرد این است که در حال حاضر امکان ذخیره یک نوع `impl Trait` در یک فیلد ساختار وجود ندارد. ممکن است روزی زمانی که [_named existential types_] به طور کامل اجرا شوند، کار کند.
This implementation is not quite optimal since it recreates the `usable_frame` allocator on every allocation. It would be better to directly store the iterator as a struct field instead. Then we wouldn't need the `nth` method and could just call [`next`] on every allocation. The problem with this approach is that it's not possible to store an `impl Trait` type in a struct field currently. It might work someday when [_named existential types_] are fully implemented.

[`next`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#tymethod.next
[_named existential types_]: https://github.com/rust-lang/rfcs/pull/2071

#### Using the `BootInfoFrameAllocator`

اکنون می‌توانیم تابع «kernel_main» خود را تغییر دهیم تا یک نمونه «BootInfoFrameAllocator» را به جای «EmptyFrameAllocator» ارسال کنیم:
We can now modify our `kernel_main` function to pass a `BootInfoFrameAllocator` instance instead of an `EmptyFrameAllocator`:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory::BootInfoFrameAllocator;
    […]
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };
    […]
}
```

با اختصاص‌دهنده فریم اطلاعات راه‌اندازی، نقشه‌برداری با موفقیت انجام می‌شود و دوباره تصویر سیاه به سفید _"New!"_ را روی صفحه می‌بینیم. در پشت صحنه، متد 'map_to' جداول صفحات گم شده را به روش زیر ایجاد می کند:
With the boot info frame allocator, the mapping succeeds and we see the black-on-white _"New!"_ on the screen again. Behind the scenes, the `map_to` method creates the missing page tables in the following way:

- یک فریم استفاده نشده از «تخصیص دهنده_فریم» تصویب شده اختصاص دهید.
- Allocate an unused frame from the passed `frame_allocator`.
- برای ایجاد یک جدول صفحه جدید و خالی، کادر را صفر کنید.
- Zero the frame to create a new, empty page table.
- ورودی جدول سطح بالاتر را به آن فریم نگاشت کنید.
- Map the entry of the higher level table to that frame.
- با سطح جدول بعدی ادامه دهید.
- Continue with the next table level.

در حالی که تابع «create_example_mapping» ما فقط چند نمونه کد است، اکنون می‌توانیم نگاشت‌های جدیدی را برای صفحات دلخواه ایجاد کنیم. این برای تخصیص حافظه یا پیاده سازی multithreading در پست های آینده ضروری خواهد بود.
While our `create_example_mapping` function is just some example code, we are now able to create new mappings for arbitrary pages. This will be essential for allocating memory or implementing multithreading in future posts.

در این مرحله، باید تابع «create_example_mapping» را دوباره حذف کنیم تا از فراخوانی تصادفی رفتار تعریف نشده، همانطور که در [بالا] توضیح داده شد، جلوگیری کنیم (#a-create-example-mapping-function).
At this point, we should delete the `create_example_mapping` function again to avoid accidentally invoking undefined behavior, as explained [above](#a-create-example-mapping-function).

## Summary

در این پست با تکنیک‌های مختلف دسترسی به فریم‌های فیزیکی جداول صفحه از جمله نگاشت هویت، نگاشت حافظه فیزیکی کامل، نگاشت موقت و جداول صفحه بازگشتی آشنا شدیم. ما تصمیم گرفتیم که حافظه فیزیکی کامل را نقشه برداری کنیم زیرا ساده، قابل حمل و قدرتمند است.
In this post we learned about different techniques to access the physical frames of page tables, including identity mapping, mapping of the complete physical memory, temporary mapping, and recursive page tables. We chose to map the complete physical memory since it's simple, portable, and powerful.

ما نمی توانیم حافظه فیزیکی را از هسته خود بدون دسترسی به جدول صفحه نقشه برداری کنیم، بنابراین به پشتیبانی بوت لودر نیاز داشتیم. جعبه «بوت‌لودر» از ایجاد نقشه‌برداری مورد نیاز از طریق ویژگی‌های بار اختیاری پشتیبانی می‌کند. اطلاعات مورد نیاز را در قالب آرگومان «&BootInfo» به تابع نقطه ورودی ما به هسته ما ارسال می کند.
We can't map the physical memory from our kernel without page table access, so we needed support from the bootloader. The `bootloader` crate supports creating the required mapping through optional cargo features. It passes the required information to our kernel in the form of a `&BootInfo` argument to our entry point function.

برای پیاده سازی خود، ابتدا جداول صفحه را به صورت دستی پیمایش کردیم تا تابع ترجمه را پیاده سازی کنیم و سپس از نوع «MappedPageTable» جعبه «x86_64» استفاده کردیم. ما همچنین یاد گرفتیم که چگونه نگاشتهای جدید را در جدول صفحه ایجاد کنیم و چگونه `FrameAllocator` لازم را در بالای نقشه حافظه ارسال شده توسط بوت لودر ایجاد کنیم.
For our implementation, we first manually traversed the page tables to implement a translation function, and then used the `MappedPageTable` type of the `x86_64` crate. We also learned how to create new mappings in the page table and how to create the necessary `FrameAllocator` on top of the memory map passed by the bootloader.

## What's next?

پست بعدی یک منطقه حافظه پشته برای هسته ما ایجاد می کند، که به ما امکان می دهد [تخصیص حافظه] و استفاده از [انواع مجموعه] مختلف.
The next post will create a heap memory region for our kernel, which will allow us to [allocate memory] and use various [collection types].

[allocate memory]: https://doc.rust-lang.org/alloc/boxed/struct.Box.html
[collection types]: https://doc.rust-lang.org/alloc/collections/index.html
