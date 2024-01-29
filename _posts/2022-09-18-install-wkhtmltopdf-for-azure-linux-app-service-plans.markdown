---
# layout: post
# author: Fati Iseni
title: "Install wkhtmltopdf for Azure Linux App Service Plans"
date: 2022-09-18 12:00:00 +0100
description: Learn how to install the wkhtmltopdf package on Azure Linux App Service Plans during app startup, enabling HTML-to-PDF and HTML-to-image conversion features for your web applications running on Linux-based hosts.
categories: [Blogging, Tutorial, Software Development]
tags: [Azure]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/pozitron-cover.png
---
When working with web applications, you may occasionally need to convert HTML content into images or PDFs. In my case, I stumbled upon this requirement and, after a quick search, I discovered a fantastic open-source package called `wkhtmltopdf`. This powerful package provides an efficient solution for such conversions, offering two command-line tools: `wkhtmltopdf` for converting HTML to PDF, and `wkhtmltoimage` for converting HTML to various image formats. You can support the authors here: https://wkhtmltopdf.org/.

While the `wkhtmltopdf` package is conveniently included in Windows plans by default, I ran into a challenge when I switched to a Linux plan. My application began throwing exceptions, and I soon realized that the tools were not pre-installed on Linux hosts. Since Azure spins up a new host/container with each application start and any data outside the `/home` directory is not persisted, just installing the package is not enough. This article will walk you through the steps to install `wkhtmltopdf` on Azure Linux App Service Plans during app startup, ensuring seamless integration with your application.

- Login to Azure Portal
- Select your App Service instance. Choose the SSH menu and then open the SSH console.

  ![Image1](/assets/img/posts/wkhtmltopdf/image-1.png)

- Create two files, `install-tools.sh` and `startup.sh`, under the `/home/site/wwwroot/` directory. Follow the steps from the screenshot below:
  - install-tools.sh
  ```sh
  apt-get update
  apt-get install -y wkhtmltopdf
  ```
  - startup.sh
  ```sh
  sh /home/site/wwwroot/install-tools.sh &
  dotnet DemoApp.dll
  ```
  ![Image2](/assets/img/posts/wkhtmltopdf/image-2.png)

- Go to your App Service instance again. Choose the `Configuration` menu, the `General settings` sub-menu, and update the startup command to `./startup.sh`. Save the changes.

  ![Image3](/assets/img/posts/wkhtmltopdf/image-3.png)

- In the Overview section of the same page, stop and start the service.
- Your application should be up and running immediately.
- After some time (it may take a couple of minutes), connect to the SSH console again and check whether the tool is available.

  ![Image4](/assets/img/posts/wkhtmltopdf/image-4.png)

I hope you found this article useful. Happy coding!