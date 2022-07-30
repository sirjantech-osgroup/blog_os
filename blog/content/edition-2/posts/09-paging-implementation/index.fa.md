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

[پست قبلی] مقدمه‌ای بر مفهوم صفحه‌بندی ارائه داد. صفحه‌بندی را با مقایسه آن با تقسیم‌بندی یک گزینه بهتر نشان داد، نحوه عملکرد صفحه‌بندی و جدول‌های صفحه را توضیح داد و سپس طراحی جدول صفحه 4 سطحی «x86_64» را معرفی کرد. ما متوجه شدیم که بوت‌لودر قبلاً یک سلسله مراتب جدول صفحه را برای هسته ما تنظیم کرده است، به این معنی که هسته ما قبلاً روی آدرس‌های مجازی اجرا می‌شود. این امر ایمنی را بهبود می‌بخشد زیرا دسترسی‌های غیرقانونی به حافظه باعث ایجاد استثناهای خطای صفحه بجای تغییر حافظه فیزیکی دلخواه می‌شود.

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

در این مثال، ما قاب‌های جدول صفحه با نگاشت هویت شده مختلف را می‌بینیم. به این ترتیب آدرس‌های فیزیکی جداول صفحه نیز آدرس‌های مجازی معتبری هستند تا بتوانیم به راحتی به جداول صفحه همه سطوح از ثبات CR3 دسترسی داشته باشیم.

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

با این حال، در x86_64، به جای صفحات پیش‌فرض 4KiB، می‌توانیم از [صفحات عظیم] با اندازه 2MiB برای نگاشت کردن استفاده کنیم. به این ترتیب، نگاشت 32GiB حافظه فیزیکی تنها به 132KiB برای جداول صفحه نیاز دارد زیرا تنها به یک جدول سطح 3 و 32 جدول سطح 2 نیاز است. صفحات بزرگ نیز از آن‌جایی که از ورودی‌های کمتری در TLB استفاده می‌کنند، در حافظه پنهان کارآمدتر هستند.

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

To construct such addresses in Rust code, you can use bitwise operations:

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

The above code assumes that the last level 4 entry with index `0o777` (511) is recursively mapped. This isn't the case currently, so the code won't work yet. See below on how to tell the bootloader to set up the recursive mapping.

Alternatively to performing the bitwise operations by hand, you can use the [`RecursivePageTable`] type of the `x86_64` crate, which provides safe abstractions for various page table operations. For example, the code below shows how to translate a virtual address to its mapped physical address:

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

Again, a valid recursive mapping is required for this code. With such a mapping, the missing `level_4_table_addr` can be calculated as in the first code example.

</details>

---

Recursive Paging is an interesting technique that shows how powerful a single mapping in a page table can be. It is relatively easy to implement and only requires a minimal amount of setup (just a single recursive entry), so it's a good choice for first experiments with paging.

However, it also has some disadvantages:

- It occupies a large amount of virtual memory (512GiB). This isn't a big problem in the large 48-bit address space, but it might lead to suboptimal cache behavior.
- It only allows accessing the currently active address space easily. Accessing other address spaces is still possible by changing the recursive entry, but a temporary mapping is required for switching back. We described how to do this in the (outdated) [_Remap The Kernel_] post.
- It heavily relies on the page table format of x86 and might not work on other architectures.

[_Remap The Kernel_]: https://os.phil-opp.com/remap-the-kernel/#overview

## Bootloader Support

All of these approaches require page table modifications for their setup. For example, mappings for the physical memory need to be created or an entry of the level 4 table needs to be mapped recursively. The problem is that we can't create these required mappings without an existing way to access the page tables.

This means that we need the help of the bootloader, which creates the page tables that our kernel runs on. The bootloader has access to the page tables, so it can create any mappings that we need. In its current implementation, the `bootloader` crate has support for two of the above approaches, controlled through [cargo features]:

[cargo features]: https://doc.rust-lang.org/cargo/reference/features.html#the-features-section

- The `map_physical_memory` feature maps the complete physical memory somewhere into the virtual address space. Thus, the kernel can access all physical memory and can follow the [_Map the Complete Physical Memory_](#map-the-complete-physical-memory) approach.
- With the `recursive_page_table` feature, the bootloader maps an entry of the level 4 page table recursively. This allows the kernel to access the page tables as described in the [_Recursive Page Tables_](#recursive-page-tables) section.

We choose the first approach for our kernel since it is simple, platform-independent, and more powerful (it also allows access to non-page-table-frames). To enable the required bootloader support, we add the `map_physical_memory` feature to our `bootloader` dependency:

```toml
[dependencies]
bootloader = { version = "0.9.8", features = ["map_physical_memory"]}
```

With this feature enabled, the bootloader maps the complete physical memory to some unused virtual address range. To communicate the virtual address range to our kernel, the bootloader passes a _boot information_ structure.

### Boot Information

The `bootloader` crate defines a [`BootInfo`] struct that contains all the information it passes to our kernel. The struct is still in an early stage, so expect some breakage when updating to future [semver-incompatible] bootloader versions. With the `map_physical_memory` feature enabled, it currently has the two fields `memory_map` and `physical_memory_offset`:

[`BootInfo`]: https://docs.rs/bootloader/0.9.3/bootloader/bootinfo/struct.BootInfo.html
[semver-incompatible]: https://doc.rust-lang.org/stable/cargo/reference/specifying-dependencies.html#caret-requirements

- The `memory_map` field contains an overview of the available physical memory. This tells our kernel how much physical memory is available in the system and which memory regions are reserved for devices such as the VGA hardware. The memory map can be queried from the BIOS or UEFI firmware, but only very early in the boot process. For this reason, it must be provided by the bootloader because there is no way for the kernel to retrieve it later. We will need the memory map later in this post.
- The `physical_memory_offset` tells us the virtual start address of the physical memory mapping. By adding this offset to a physical address, we get the corresponding virtual address. This allows us to access arbitrary physical memory from our kernel.

The bootloader passes the `BootInfo` struct to our kernel in the form of a `&'static BootInfo` argument to our `_start` function. We don't have this argument declared in our function yet, so let's add it:

```rust
// in src/main.rs

use bootloader::BootInfo;

#[no_mangle]
pub extern "C" fn _start(boot_info: &'static BootInfo) -> ! { // new argument
    […]
}
```

It wasn't a problem to leave off this argument before because the x86_64 calling convention passes the first argument in a CPU register. Thus, the argument is simply ignored when it isn't declared. However, it would be a problem if we accidentally used a wrong argument type, since the compiler doesn't know the correct type signature of our entry point function.

### The `entry_point` Macro

Since our `_start` function is called externally from the bootloader, no checking of our function signature occurs. This means that we could let it take arbitrary arguments without any compilation errors, but it would fail or cause undefined behavior at runtime.

To make sure that the entry point function has always the correct signature that the bootloader expects, the `bootloader` crate provides an [`entry_point`] macro that provides a type-checked way to define a Rust function as the entry point. Let's rewrite our entry point function to use this macro:

[`entry_point`]: https://docs.rs/bootloader/0.6.4/bootloader/macro.entry_point.html

```rust
// in src/main.rs

use bootloader::{BootInfo, entry_point};

entry_point!(kernel_main);

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
}
```

We no longer need to use `extern "C"` or `no_mangle` for our entry point, as the macro defines the real lower level `_start` entry point for us. The `kernel_main` function is now a completely normal Rust function, so we can choose an arbitrary name for it. The important thing is that it is type-checked so that a compilation error occurs when we use a wrong function signature, for example by adding an argument or changing the argument type.

Let's perform the same change in our `lib.rs`:

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

Since the entry point is only used in test mode, we add the `#[cfg(test)]` attribute to all items. We give our test entry point the distinct name `test_kernel_main` to avoid confusion with the `kernel_main` of our `main.rs`. We don't use the `BootInfo` parameter for now, so we prefix the parameter name with a `_` to silence the unused variable warning.

## Implementation

Now that we have access to physical memory, we can finally start to implement our page table code. First, we will take a look at the currently active page tables that our kernel runs on. In the second step, we will create a translation function that returns the physical address that a given virtual address is mapped to. As the last step, we will try to modify the page tables in order to create a new mapping.

Before we begin, we create a new `memory` module for our code:

```rust
// in src/lib.rs

pub mod memory;
```

For the module we create an empty `src/memory.rs` file.

### Accessing the Page Tables

At the [end of the previous post], we tried to take a look at the page tables our kernel runs on, but failed since we couldn't access the physical frame that the `CR3` register points to. We're now able to continue from there by creating an `active_level_4_table` function that returns a reference to the active level 4 page table:

[end of the previous post]: @/edition-2/posts/08-paging-introduction/index.md#accessing-the-page-tables

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

First, we read the physical frame of the active level 4 table from the `CR3` register. We then take its physical start address, convert it to an `u64`, and add it to `physical_memory_offset` to get the virtual address where the page table frame is mapped. Finally, we convert the virtual address to a `*mut PageTable` raw pointer through the `as_mut_ptr` method and then unsafely create a `&mut PageTable` reference from it. We create a `&mut` reference instead of a `&` reference because we will mutate the page tables later in this post.

We don't need to use an unsafe block here because Rust treats the complete body of an `unsafe fn` like a large `unsafe` block. This makes our code more dangerous since we could accidentally introduce an unsafe operation in previous lines without noticing. It also makes it much more difficult to spot the unsafe operations. There is an [RFC](https://github.com/rust-lang/rfcs/pull/2585) to change this behavior.

We can now use this function to print the entries of the level 4 table:

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

First, we convert the `physical_memory_offset` of the `BootInfo` struct to a [`VirtAddr`] and pass it to the `active_level_4_table` function. We then use the `iter` function to iterate over the page table entries and the [`enumerate`] combinator to additionally add an index `i` to each element. We only print non-empty entries because all 512 entries wouldn't fit on the screen.

[`VirtAddr`]: https://docs.rs/x86_64/0.14.2/x86_64/addr/struct.VirtAddr.html
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate

When we run it, we see the following output:

![QEMU printing entry 0 (0x2000, PRESENT, WRITABLE, ACCESSED), entry 1 (0x894000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 31 (0x88e000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 175 (0x891000, PRESENT, WRITABLE, ACCESSED, DIRTY), and entry 504 (0x897000, PRESENT, WRITABLE, ACCESSED, DIRTY)](qemu-print-level-4-table.png)

We see that there are various non-empty entries, which all map to different level 3 tables. There are so many regions because kernel code, kernel stack, the physical memory mapping, and the boot information all use separate memory areas.

To traverse the page tables further and take a look at a level 3 table, we can take the mapped frame of an entry and convert it to a virtual address again:

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

For looking at the level 2 and level 1 tables, we repeat that process for the level 3 and level 2 entries. As you can imagine, this gets very verbose quickly, so we don't show the full code here.

Traversing the page tables manually is interesting because it helps to understand how the CPU performs the translation. However, most of the time we are only interested in the mapped physical address for a given virtual address, so let's create a function for that.

### Translating Addresses

For translating a virtual to a physical address, we have to traverse the four-level page table until we reach the mapped frame. Let's create a function that performs this translation:

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

We forward the function to a safe `translate_addr_inner` function to limit the scope of `unsafe`. As we noted above, Rust treats the complete body of an unsafe fn like a large unsafe block. By calling into a private safe function, we make each `unsafe` operation explicit again.

The private inner function contains the real implementation:

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

Instead of reusing our `active_level_4_table` function, we read the level 4 frame from the `CR3` register again. We do this because it simplifies this prototype implementation. Don't worry, we will create a better solution in a moment.

The `VirtAddr` struct already provides methods to compute the indexes into the page tables of the four levels. We store these indexes in a small array because it allows us to traverse the page tables using a `for` loop. Outside of the loop, we remember the last visited `frame` to calculate the physical address later. The `frame` points to page table frames while iterating, and to the mapped frame after the last iteration, i.e. after following the level 1 entry.

Inside the loop, we again use the `physical_memory_offset` to convert the frame into a page table reference. We then read the entry of the current page table and use the [`PageTableEntry::frame`] function to retrieve the mapped frame. If the entry is not mapped to a frame we return `None`. If the entry maps a huge 2MiB or 1GiB page we panic for now.

[`PageTableEntry::frame`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/page_table/struct.PageTableEntry.html#method.frame

Let's test our translation function by translating some addresses:

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

When we run it, we see the following output:

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, "panicked at 'huge pages not supported'](qemu-translate-addr.png)

As expected, the identity-mapped address `0xb8000` translates to the same physical address. The code page and the stack page translate to some arbitrary physical addresses, which depend on how the bootloader created the initial mapping for our kernel. It's worth noting that the last 12 bits always stay the same after translation, which makes sense because these bits are the [_page offset_] and not part of the translation.

[_page offset_]: @/edition-2/posts/08-paging-introduction/index.md#paging-on-x86-64

Since each physical address can be accessed by adding the `physical_memory_offset`, the translation of the `physical_memory_offset` address itself should point to physical address `0`. However, the translation fails because the mapping uses huge pages for efficiency, which is not supported in our implementation yet.

### Using `OffsetPageTable`

Translating virtual to physical addresses is a common task in an OS kernel, therefore the `x86_64` crate provides an abstraction for it. The implementation already supports huge pages and several other page table functions apart from `translate_addr`, so we will use it in the following instead of adding huge page support to our own implementation.

The base of the abstraction are two traits that define various page table mapping functions:

- The [`Mapper`] trait is generic over the page size and provides functions that operate on pages. Examples are [`translate_page`], which translates a given page to a frame of the same size, and [`map_to`], which creates a new mapping in the page table.
- The [`Translate`] trait provides functions that work with multiple page sizes such as [`translate_addr`] or the general [`translate`].

[`Mapper`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html
[`translate_page`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html#tymethod.translate_page
[`map_to`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Mapper.html#method.map_to
[`Translate`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html
[`translate_addr`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html#method.translate_addr
[`translate`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/trait.Translate.html#tymethod.translate

The traits only define the interface, they don't provide any implementation. The `x86_64` crate currently provides three types that implement the traits with different requirements. The [`OffsetPageTable`] type assumes that the complete physical memory is mapped to the virtual address space at some offset. The [`MappedPageTable`] is a bit more flexible: It only requires that each page table frame is mapped to the virtual address space at a calculable address. Finally, the [`RecursivePageTable`] type can be used to access page table frames through [recursive page tables](#recursive-page-tables).

[`OffsetPageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.OffsetPageTable.html
[`MappedPageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MappedPageTable.html
[`RecursivePageTable`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.RecursivePageTable.html

In our case, the bootloader maps the complete physical memory at a virtual address specified by the `physical_memory_offset` variable, so we can use the `OffsetPageTable` type. To initialize it, we create a new `init` function in our `memory` module:

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

The function takes the `physical_memory_offset` as an argument and returns a new `OffsetPageTable` instance with a `'static` lifetime. This means that the instance stays valid for the complete runtime of our kernel. In the function body, we first call the `active_level_4_table` function to retrieve a mutable reference to the level 4 page table. We then invoke the [`OffsetPageTable::new`] function with this reference. As the second parameter, the `new` function expects the virtual address at which the mapping of the physical memory starts, which is given in the `physical_memory_offset` variable.

[`OffsetPageTable::new`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.OffsetPageTable.html#method.new

The `active_level_4_table` function should be only called from the `init` function from now on because it can easily lead to aliased mutable references when called multiple times, which can cause undefined behavior. For this reason, we make the function private by removing the `pub` specifier.

We now can use the `Translate::translate_addr` method instead of our own `memory::translate_addr` function. We only need to change a few lines in our `kernel_main`:

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

We need to import the `Translate` trait in order to use the [`translate_addr`] method it provides.

When we run it now, we see the same translation results as before, with the difference that the huge page translation now also works:

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, 0x18000000000 -> 0x0](qemu-mapper-translate-addr.png)

As expected, the translations of `0xb8000` and the code and stack addresses stay the same as with our own translation function. Additionally, we now see that the virtual address `physical_memory_offset` is mapped to the physical address `0x0`.

By using the translation function of the `MappedPageTable` type we can spare ourselves the work of implementing huge page support. We also have access to other page functions such as `map_to`, which we will use in the next section.

At this point we no longer need our `memory::translate_addr` and `memory::translate_addr_inner` functions, so we can delete them.

### Creating a new Mapping

Until now we only looked at the page tables without modifying anything. Let's change that by creating a new mapping for a previously unmapped page.

We will use the [`map_to`] function of the [`Mapper`] trait for our implementation, so let's take a look at that function first. The documentation tells us that it takes four arguments: the page that we want to map, the frame that the page should be mapped to, a set of flags for the page table entry, and a `frame_allocator`. The frame allocator is needed because mapping the given page might require creating additional page tables, which need unused frames as backing storage.

[`map_to`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.Mapper.html#tymethod.map_to
[`Mapper`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.Mapper.html

#### A `create_example_mapping` Function

The first step of our implementation is to create a new `create_example_mapping` function that maps a given virtual page to `0xb8000`, the physical frame of the VGA text buffer. We choose that frame because it allows us to easily test if the mapping was created correctly: We just need to write to the newly mapped page and see whether we see the write appear on the screen.

The `create_example_mapping` function looks like this:

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

In addition to the `page` that should be mapped, the function expects a mutable reference to an `OffsetPageTable` instance and a `frame_allocator`. The `frame_allocator` parameter uses the [`impl Trait`][impl-trait-arg] syntax to be [generic] over all types that implement the [`FrameAllocator`] trait. The trait is generic over the [`PageSize`] trait to work with both standard 4KiB pages and huge 2MiB/1GiB pages. We only want to create a 4KiB mapping, so we set the generic parameter to `Size4KiB`.

[impl-trait-arg]: https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters
[generic]: https://doc.rust-lang.org/book/ch10-00-generics.html
[`FrameAllocator`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/trait.FrameAllocator.html
[`PageSize`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/page/trait.PageSize.html

The [`map_to`] method is unsafe because the caller must ensure that the frame is not already in use. The reason for this is that mapping the same frame twice could result in undefined behavior, for example when two different `&mut` references point to the same physical memory location. In our case, we reuse the VGA text buffer frame, which is already mapped, so we break the required condition. However, the `create_example_mapping` function is only a temporary testing function and will be removed after this post, so it is ok. To remind us of the unsafety, we put a `FIXME` comment on the line.

In addition to the `page` and the `unused_frame`, the `map_to` method takes a set of flags for the mapping and a reference to the `frame_allocator`, which will be explained in a moment. For the flags, we set the `PRESENT` flag because it is required for all valid entries and the `WRITABLE` flag to make the mapped page writable. For a list of all possible flags, see the [_Page Table Format_] section of the previous post.

[_Page Table Format_]: @/edition-2/posts/08-paging-introduction/index.md#page-table-format

The [`map_to`] function can fail, so it returns a [`Result`]. Since this is just some example code that does not need to be robust, we just use [`expect`] to panic when an error occurs. On success, the function returns a [`MapperFlush`] type that provides an easy way to flush the newly mapped page from the translation lookaside buffer (TLB) with its [`flush`] method. Like `Result`, the type uses the [`#[must_use]`][must_use] attribute to emit a warning when we accidentally forget to use it.

[`Result`]: https://doc.rust-lang.org/core/result/enum.Result.html
[`expect`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.expect
[`MapperFlush`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MapperFlush.html
[`flush`]: https://docs.rs/x86_64/0.14.2/x86_64/structures/paging/mapper/struct.MapperFlush.html#method.flush
[must_use]: https://doc.rust-lang.org/std/result/#results-must-be-used

#### A dummy `FrameAllocator`

To be able to call `create_example_mapping` we need to create a type that implements the `FrameAllocator` trait first. As noted above, the trait is responsible for allocating frames for new page table if they are needed by `map_to`.

Let's start with the simple case and assume that we don't need to create new page tables. For this case, a frame allocator that always returns `None` suffices. We create such an `EmptyFrameAllocator` for testing our mapping function:

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

Implementing the `FrameAllocator` is unsafe because the implementer must guarantee that the allocator yields only unused frames. Otherwise undefined behavior might occur, for example when two virtual pages are mapped to the same physical frame. Our `EmptyFrameAllocator` only returns `None`, so this isn't a problem in this case.

#### Choosing a Virtual Page

We now have a simple frame allocator that we can pass to our `create_example_mapping` function. However, the allocator always returns `None`, so this will only work if no additional page table frames are needed for creating the mapping. To understand when additional page table frames are needed and when not, let's consider an example:

![A virtual and a physical address space with a single mapped page and the page tables of all four levels](required-page-frames-example.svg)

The graphic shows the virtual address space on the left, the physical address space on the right, and the page tables in between. The page tables are stored in physical memory frames, indicated by the dashed lines. The virtual address space contains a single mapped page at address `0x803fe00000`, marked in blue. To translate this page to its frame, the CPU walks the 4-level page table until it reaches the frame at address 36 KiB.

Additionally, the graphic shows the physical frame of the VGA text buffer in red. Our goal is to map a previously unmapped virtual page to this frame using our `create_example_mapping` function. Since our `EmptyFrameAllocator` always returns `None`, we want to create the mapping so that no additional frames are needed from the allocator. This depends on the virtual page that we select for the mapping.

The graphic shows two candidate pages in the virtual address space, both marked in yellow. One page is at address `0x803fdfd000`, which is 3 pages before the mapped page (in blue). While the level 4 and level 3 page table indices are the same as for the blue page, the level 2 and level 1 indices are different (see the [previous post][page-table-indices]). The different index into the level 2 table means that a different level 1 table is used for this page. Since this level 1 table does not exist yet, we would need to create it if we chose that page for our example mapping, which would require an additional unused physical frame. In contrast, the second candidate page at address `0x803fe02000` does not have this problem because it uses the same level 1 page table than the blue page. Thus, all required page tables already exist.

[page-table-indices]: @/edition-2/posts/08-paging-introduction/index.md#paging-on-x86-64

In summary, the difficulty of creating a new mapping depends on the virtual page that we want to map. In the easiest case, the level 1 page table for the page already exists and we just need to write a single entry. In the most difficult case, the page is in a memory region for that no level 3 exists yet so that we need to create new level 3, level 2 and level 1 page tables first.

For calling our `create_example_mapping` function with the `EmptyFrameAllocator`, we need to choose a page for that all page tables already exist. To find such a page, we can utilize the fact that the bootloader loads itself in the first megabyte of the virtual address space. This means that a valid level 1 table exists for all pages this region. Thus, we can choose any unused page in this memory region for our example mapping, such as the page at address `0`. Normally, this page should stay unused to guarantee that dereferencing a null pointer causes a page fault, so we know that the bootloader leaves it unmapped.

#### Creating the Mapping

We now have all the required parameters for calling our `create_example_mapping` function, so let's modify our `kernel_main` function to map the page at virtual address `0`. Since we map the page to the frame of the VGA text buffer, we should be able to write to the screen through it afterwards. The implementation looks like this:

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

We first create the mapping for the page at address `0` by calling our `create_example_mapping` function with a mutable reference to the `mapper` and the `frame_allocator` instances. This maps the page to the VGA text buffer frame, so we should see any write to it on the screen.

Then we convert the page to a raw pointer and write a value to offset `400`. We don't write to the start of the page because the top line of the VGA buffer is directly shifted off the screen by the next `println`. We write the value `0x_f021_f077_f065_f04e`, which represents the string _"New!"_ on white background. As we learned [in the _“VGA Text Mode”_ post], writes to the VGA buffer should be volatile, so we use the [`write_volatile`] method.

[in the _“VGA Text Mode”_ post]: @/edition-2/posts/03-vga-text-buffer/index.md#volatile
[`write_volatile`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write_volatile

When we run it in QEMU, we see the following output:

![QEMU printing "It did not crash!" with four completely white cells in the middle of the screen](qemu-new-mapping.png)

The _"New!"_ on the screen is by our write to page `0`, which means that we successfully created a new mapping in the page tables.

Creating that mapping only worked because the level 1 table responsible for the page at address `0` already exists. When we try to map a page for that no level 1 table exists yet, the `map_to` function fails because it tries to allocate frames from the `EmptyFrameAllocator` for creating new page tables. We can see that happen when we try to map page `0xdeadbeaf000` instead of `0`:

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
    let page = Page::containing_address(VirtAddr::new(0xdeadbeaf000));
    […]
}
```

When we run it, a panic with the following error message occurs:

```
panicked at 'map_to failed: FrameAllocationFailed', /…/result.rs:999:5
```

To map pages that don't have a level 1 page table yet we need to create a proper `FrameAllocator`. But how do we know which frames are unused and how much physical memory is available?

### Allocating Frames

In order to create new page tables, we need to create a proper frame allocator. For that we use the `memory_map` that is passed by the bootloader as part of the `BootInfo` struct:

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

The struct has two fields: A `'static` reference to the memory map passed by the bootloader and a `next` field that keeps track of number of the next frame that the allocator should return.

As we explained in the [_Boot Information_](#boot-information) section, the memory map is provided by the BIOS/UEFI firmware. It can only be queried very early in the boot process, so the bootloader already calls the respective functions for us. The memory map consists of a list of [`MemoryRegion`] structs, which contain the start address, the length, and the type (e.g. unused, reserved, etc.) of each memory region.

The `init` function initializes a `BootInfoFrameAllocator` with a given memory map. The `next` field is initialized with `0` and will be increased for every frame allocation to avoid returning the same frame twice. Since we don't know if the usable frames of the memory map were already used somewhere else, our `init` function must be `unsafe` to require additional guarantees from the caller.

#### A `usable_frames` Method

Before we implement the `FrameAllocator` trait, we add an auxiliary method that converts the memory map into an iterator of usable frames:

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

This function uses iterator combinator methods to transform the initial `MemoryMap` into an iterator of usable physical frames:

- First, we call the `iter` method to convert the memory map to an iterator of [`MemoryRegion`]s.
- Then we use the [`filter`] method to skip any reserved or otherwise unavailable regions. The bootloader updates the memory map for all the mappings it creates, so frames that are used by our kernel (code, data or stack) or to store the boot information are already marked as `InUse` or similar. Thus we can be sure that `Usable` frames are not used somewhere else.
- Afterwards, we use the [`map`] combinator and Rust's [range syntax] to transform our iterator of memory regions to an iterator of address ranges.
- Next, we use [`flat_map`] to transform the address ranges into an iterator of frame start addresses, choosing every 4096th address using [`step_by`]. Since 4096 bytes (= 4 KiB) is the page size, we get the start address of each frame. The bootloader page aligns all usable memory areas so that we don't need any alignment or rounding code here. By using [`flat_map`] instead of `map`, we get an `Iterator<Item = u64>` instead of an `Iterator<Item = Iterator<Item = u64>>`.
- Finally, we convert the start addresses to `PhysFrame` types to construct the an `Iterator<Item = PhysFrame>`.

[`MemoryRegion`]: https://docs.rs/bootloader/0.6.4/bootloader/bootinfo/struct.MemoryRegion.html
[`filter`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.filter
[`map`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.map
[range syntax]: https://doc.rust-lang.org/core/ops/struct.Range.html
[`step_by`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.step_by
[`flat_map`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.flat_map

The return type of the function uses the [`impl Trait`] feature. This way, we can specify that we return some type that implements the [`Iterator`] trait with item type `PhysFrame`, but don't need to name the concrete return type. This is important here because we _can't_ name the concrete type since it depends on unnamable closure types.

[`impl Trait`]: https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits
[`Iterator`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html

#### Implementing the `FrameAllocator` Trait

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

We first use the `usable_frames` method to get an iterator of usable frames from the memory map. Then, we use the [`Iterator::nth`] function to get the frame with index `self.next` (thereby skipping `(self.next - 1)` frames). Before returning that frame, we increase `self.next` by one so that we return the following frame on the next call.

[`Iterator::nth`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.nth

This implementation is not quite optimal since it recreates the `usable_frame` allocator on every allocation. It would be better to directly store the iterator as a struct field instead. Then we wouldn't need the `nth` method and could just call [`next`] on every allocation. The problem with this approach is that it's not possible to store an `impl Trait` type in a struct field currently. It might work someday when [_named existential types_] are fully implemented.

[`next`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#tymethod.next
[_named existential types_]: https://github.com/rust-lang/rfcs/pull/2071

#### Using the `BootInfoFrameAllocator`

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

With the boot info frame allocator, the mapping succeeds and we see the black-on-white _"New!"_ on the screen again. Behind the scenes, the `map_to` method creates the missing page tables in the following way:

- Allocate an unused frame from the passed `frame_allocator`.
- Zero the frame to create a new, empty page table.
- Map the entry of the higher level table to that frame.
- Continue with the next table level.

While our `create_example_mapping` function is just some example code, we are now able to create new mappings for arbitrary pages. This will be essential for allocating memory or implementing multithreading in future posts.

At this point, we should delete the `create_example_mapping` function again to avoid accidentally invoking undefined behavior, as explained [above](#a-create-example-mapping-function).

## Summary

In this post we learned about different techniques to access the physical frames of page tables, including identity mapping, mapping of the complete physical memory, temporary mapping, and recursive page tables. We chose to map the complete physical memory since it's simple, portable, and powerful.

We can't map the physical memory from our kernel without page table access, so we needed support from the bootloader. The `bootloader` crate supports creating the required mapping through optional cargo features. It passes the required information to our kernel in the form of a `&BootInfo` argument to our entry point function.

For our implementation, we first manually traversed the page tables to implement a translation function, and then used the `MappedPageTable` type of the `x86_64` crate. We also learned how to create new mappings in the page table and how to create the necessary `FrameAllocator` on top of the memory map passed by the bootloader.

## What's next?

The next post will create a heap memory region for our kernel, which will allow us to [allocate memory] and use various [collection types].

[allocate memory]: https://doc.rust-lang.org/alloc/boxed/struct.Box.html
[collection types]: https://doc.rust-lang.org/alloc/collections/index.html
