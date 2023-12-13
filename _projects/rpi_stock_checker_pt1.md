---
layout: page
title: Raspberry Pi Stock Checker
date: 2023-12-02 08:00:00-0000
description: Simple CLI to periodically check RSS feeds for Raspberry Pi 5 stock.
img: assets/img/win_notification.png
importance: 1
category: RPi Stock Checker
giscus_comments: true
giscus_repo: aven-arlington/rpi-stock-checker
toc:
  sidebar: left
---
## Background
The Raspberry Pi 5 has begun shipping to distributors across the globe and I am excited to get my hands on one.
Despite having signed up for a pre-order in October, I still hadn't made it to the front of the back-order line. 
With Christmas fast approaching I needed to track down some stock and order one.

I found myself checking the amazingly useful [rpilocator.com](https://rpilocator.com/) once a day or so and thought
"wouldn't it be nice if I could automate this check and get a notification?". I started looking into it and sure
enough, the folks at [rpilocator.com]([https://rpilocator.com/) offer a [RSS feed](https://rpilocator.com/feed/)!

The next step was to figure out how to download and parse a RSS feed. Luckily a quick google search turned up
a the [Rust for Windows API](https://learn.microsoft.com/en-us/windows/dev-environment/rust/) which includes 
a fully functional [RSS reader](https://learn.microsoft.com/en-us/windows/dev-environment/rust/rss-reader-rust-for-windows) example.

## Prototyping
I created the rpi_stock_checker repository on GitHub and started configuring the Cargo.toml according to the example.
```toml
[dependencies.windows]
version = "*"
features = [
   "Foundation_Collections",
   "Web_Syndication",
   "UI_Notifications",
   "Data_Xml_Dom",
]
```

Next I update src/main.rs with the basic example code for testing I made sure
to update the feed url to point to [rpilocator.com/feed](https://rpilocator.com/feed/) as well.
```rust
use windows::{
    core::*,
    Foundation::Uri,
    Web::Syndication::SyndicationClient
};

fn main() -> windows::core::Result<()> {
    let uri = Uri::CreateUri(h!("https://rpilocator.com/feed/"))?;
    let client = SyndicationClient::new()?;

    client.SetRequestHeader(
        h!("User-Agent"),
        h!("Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; WOW64; Trident/6.0)"),
    )?;

    let feed = client.RetrieveFeedAsync(&uri)?.get()?;

    for item in feed.Items()? {
        println!("{}", item.Title()?.Text()?);
    }

    Ok(())
}
```

Executing the program results in the the output which in turn matches the live website.
That means we now have feed data to work with.
```cmd
PS C:\rpi-stock-checker> cargo run
   Compiling rpi-stock-checker v0.1.0 (C:\rpi-stock-checker)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27s
     Running `target\debug\rpi-stock-checker.exe`
Stock Alert (PL): RPi 4 Model B - 8GB RAM is In Stock at Botland
Stock Alert (US): RPi CM4 - 4GB RAM, 8GB MMC, With Wifi is In Stock at Pishop 250 units in stock.
Stock Alert (US): RPi CM4 - 8GB RAM, 16GB MMC, With Wifi is In Stock at Pishop 250 units in stock.
Stock Alert (US): RPi CM4 - 8GB RAM, No MMC, No Wifi is In Stock at Pishop 250 units in stock.
```

## Parsing the Feed
Having the feed data is great, but our main goal is to have the program search the feed for the
item we are interested in.

Lets start with some refactoring. When we call RetrieveFeedAsync there is a
possibility that the call will fail if, for example, if the internet connection is interrupted.
We want our program to be running in the background for long period of time and will need to
handle the occasional connection hiccup gracefully. To facilitate handling the types of errors
we expect to encounter the most often, we will have a separate function for checking the feed.

The check_feed function does all the heavy lifting to pull the RSS feed. We pass in a reference
to the SyndicationClient and call RetrieveFeedAsync with it. If the connection fails, we return
a Err result. If the connection is successful, we create a Vector of Strings to hold the feeds
and return that to the caller.
```rust
fn check_feed(client: &SyndicationClient) -> Result<Vec<String>> {
    let uri = Uri::CreateUri(h!("https://rpilocator.com/feed"))?;
    let feed = client.RetrieveFeedAsync(&uri)?.get()?;
    let feed_items: IVector<SyndicationItem> = feed.Items()?;
    let mut output_strings: Vec<String> = Vec::new();
    for item in feed_items {
        output_strings.push(item.Title().ok().unwrap().Text().ok().unwrap().to_string());
    }
    Ok(output_strings)
}
```

This approach allows us to branch execution based on a success or failure. If we get a failure
we should simply print a courtesy string to stdout and then wait a few minutes to check again.
```rust
loop {
    match check_feed(&client) {
        Ok(new_feeds) => {
            // Do stuff //
        }
        Err(_e) => {
            println!("Check_feed failed. Retrying in 5 minutes")
        }
    }
    // Update the stdout to show program is not idle/hung
    println!("Sleeping for 5 minutes");
    // Only check every 5 minutes to avoid spamming the RSS feed
    thread::sleep(Duration::from_secs(300));
}
```

Now that we have feed data coming in every 5 minutes, we need to filter out the feed items we have already
seen and move on to the next. Items that haven't been seen before are printed to stdout with a timestamp
so we can tell our program is running.
```rust
Ok(new_feeds) => {
    for feed in new_feeds {
        if !prev_feeds.contains(&feed) {
            // New feed found
            // Do something with the new feed
            prev_feeds.insert(feed);
        } else {
            continue;
        }
    }
}
```

Now we want to send a Windows Toast notification when we find a product that matches our search string.
We create another standalone function to handle the notification process and pass it the new feed item.
The notify function then checks for the presence of our search string and it it matches, will initiate
the Windows Toast Notification. 

The process of creating and sending a notification involves creating ToastNotification object and then
showing it. Notifications are built from XML so we create a XmlDocument from a template, and then update
the "text" node in that document with a new XmlText item that contains our feed item string.
```rust
fn notify(feed: &String) -> Result<()> {
    if feed.contains(SEARCH_STRING) {
        // New feed detects search string. Send toast notification
        let notification = {
            let toast_xml: XmlDocument =
                ToastNotificationManager::GetTemplateContent(ToastTemplateType::ToastText01)?;
            let text_node: IXmlNode = toast_xml.GetElementsByTagName(h!("text"))?.Item(0)?;
            let text: XmlText = toast_xml.CreateTextNode(&HSTRING::from(feed))?;
            text_node.AppendChild(&text)?;
            ToastNotification::CreateToastNotification(&toast_xml)?
        };
        // Send the toast notification
    }
    Ok(())
}
```


Once we have a ToastNotification object with our feed item text, we provide it to the ToastNotificationManager
to handle showing it. Note: showing the notification requires the AUMID from the application running the
executable. In my case it is PowerShell 7.0. More on that below...
```rust
ToastNotificationManager::GetDefault()?
            .CreateToastNotifierWithId(&HSTRING::from(AUMID))?
            .Show(&notification)?;
```

In order for the Windows Toast Notifications to work properly, the application must be built
with the correct Application User Model ID (AUMID) for the terminal executing the application.
The AUID may vary from machine to machine so this value must be set manually.

To find the AUMID for PowerShell or Windows Terminal, run the following command in PowerShell:
```console
> Get-StartApps powershell
```

And then copy the AppID value from the output into the AUMID constant in src/main.rs:line21

For example:
My machine produces the following output for Get-StartApps:
```console
PS C:\rpi-stock-checker> Get-StartApps powershell
```

## Wrapping Up
Once we have all these pieces in place, main.rs should look something like the code below.
The application gets the RSS feeds from the source, removes duplicates, and searches them
for the product we are interested in. The application is fault tolerant of network outages
and will re-try failed connections. And finally, if our product of interest is found, we get
a windows notification to let us know it is in stock somewhere!

## Next Steps
There is still more to improve! To take the app even further, the next installment will
break our code into manageable modules and add some unit testing.

## Code
[Source Code on GitHub](https://github.com/aven-arlington/rpi-stock-checker/tree/v0.1)
```rust
use chrono::prelude::Local;
use std::collections::HashSet;
use std::thread;
use std::time::Duration;
use windows::{
    core::*,
    Data::Xml::Dom::{IXmlNode, XmlDocument, XmlText},
    Foundation::{Collections::IVector, Uri},
    Web::Syndication::{SyndicationClient, SyndicationItem},
    UI::Notifications::{ToastNotification, ToastNotificationManager, ToastTemplateType},
};

// Modify this search string to math the product you are looking for
static SEARCH_STRING: &str = "(US): RPi 5 - 8GB RAM";

// Use the Microsoft AUMID for powershell, or terminal for notifications to
// function properly
// To find the AUMID of your powershell or terminal instance, run the following
// in powershell:
// Get-StartApps powershell
static AUMID: &str = "Microsoft.AutoGenerated.{A49227EA-5AF0-D494-A3F1-0918A278ED71}";

fn main() -> Result<()> {
    // Setup the RSS feed reader
    let mut prev_feeds: HashSet<String> = HashSet::new();
    let client = SyndicationClient::new()?;

    loop {
        // Get available feeds
        client.SetRequestHeader(
            h!("User-Agent"),
            h!("Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; WOW64; Trident/6.0)"),
        )?;

        match check_feed(&client) {
            Ok(new_feeds) => {
                for feed in new_feeds {
                    if !prev_feeds.contains(&feed) {
                        // New feed found
                        let now = Local::now();
                        println!("{} - {}", now.format("%Y-%m-%d %H:%M:%S"), feed);
                        notify(&feed).expect("Failed to send notification");
                        prev_feeds.insert(feed);
                    } else {
                        continue;
                    }
                    // Allow time for async toast creation to complete
                    thread::sleep(Duration::from_millis(500));
                }
            }
            Err(_e) => {
                println!("Check_feed failed. Retrying in 5 minutes")
            }
        };

        // Update the stdout to show program is not idle/hung
        println!("Sleeping for 5 minutes");
        // Only check every 5 minutes to avoid spamming the RSS feed
        thread::sleep(Duration::from_secs(300));
    }
}

fn check_feed(client: &SyndicationClient) -> Result<Vec<String>> {
    let uri = Uri::CreateUri(h!("https://rpilocator.com/feed"))?;
    let feed = client.RetrieveFeedAsync(&uri)?.get()?;
    let feed_items: IVector<SyndicationItem> = feed.Items()?;
    let mut output_strings: Vec<String> = Vec::new();
    for item in feed_items {
        output_strings.push(item.Title().ok().unwrap().Text().ok().unwrap().to_string());
    }
    Ok(output_strings)
}

fn notify(feed: &String) -> Result<()> {
    if feed.contains(SEARCH_STRING) {
        // New feed detects search string. Send toast notification
        let notification = {
            let toast_xml: XmlDocument =
                ToastNotificationManager::GetTemplateContent(ToastTemplateType::ToastText01)?;
            let text_node: IXmlNode = toast_xml.GetElementsByTagName(h!("text"))?.Item(0)?;
            let text: XmlText = toast_xml.CreateTextNode(&HSTRING::from(feed))?;
            text_node.AppendChild(&text)?;
            ToastNotification::CreateToastNotification(&toast_xml)?
        };

        // Send the toast notification
        ToastNotificationManager::GetDefault()?
            .CreateToastNotifierWithId(&HSTRING::from(AUMID))?
            .Show(&notification)?;
    }
    Ok(())
}
```

