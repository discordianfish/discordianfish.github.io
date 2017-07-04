---
title: 'Frontend Testing Zoo - Or, Nightwatch without Selenium'
date: 2017-04-17 12:50:12 +0000
tags: 
layout: post
---
What I always find hardest to figure it when it comes to unknown systems, it’s ‘how things fit together’. These project might have great documentation, but how the ecosystem plays together is usually left as an exercise for the reader.
This happened to me again when diving into frontend testing. I figured out how to get nightwatch+selenium-standalone+chromedriver+chrome working somehow, without fully understanding each part in the puzzle. I went with Nightwatch because I like the syntax and it seems one of the obvious options nowadays. That works until I updated some dependencies and broke everything again. So I took some time to figure out how these things fit together and what is really needed for me tests.
I’ve learned that Selenium is the godmother of modern frontend testing. It’s a framework with bindings for multiple languages. To decouple the server from the actual browsers, it invented “Selenium RC” as a protocol. That lead to the development of the WebDriver W3C standard which replaces Selenium RC is newer versions and is implemented for the most common browsers (Chrome -> Chromedriver, Firefox -> FirefoxDriver).
## Nightwatch without Selenium
Apparently WebDriver is also used for talking *to* Selenium. I think this is called ‘hub’ or something, that’s about as deep as my ‘deep dive’ gets for now. Either way, this makes the big picture look like:
```
Nightwatch -[webdriver]-> Selenium -[webdriver]-> Chromedriver -> Chrome
```
Selenium comes in a server and standalone flavor, where the server apparently is only used for grid based testing where you would have a bunch of Selenium nodes running on different systems. Turns out, if you don’t need grid based testing or support for legacy browsers which don’t provide WebDriver adapter, there is little use for Selenium after all.
Fortunately you can make Nightwatch talk directly to Chromedriver. This requires start_processto be set to false and selenium_host/_port to point the address of Chromedriver:
```
{
  "output_folder": false,
  "src_folders": ["tests"],
  "selenium": {
    "start_process": false
  },
  "test_settings": {
    "default": {
      "selenium_host": "127.0.0.1",
      "selenium_port": "9515",
      "screenshots": {
        "enabled": true,
        "on_failure": true,
        "on_error" : true,
        "path": "results/screenshots"
      },
      "desiredCapabilities": {
        "browserName": "chrome",
        "javascriptEnabled" : true,
        "acceptSslCerts" : true,
        "chromeOptions" : {
          "args": [
            "--no-sandbox",
            "start-fullscreen",
            "window-size=1280,800"
          ]
        }
      },
      "launch_url": "http://app"
    }
  }
}
```
The only caveat being, you need to run Chromedriver manually. I run all this in a Docker container with a entrypoint like this:
```
Xvfb :23 &
chromedriver --port=9515 --url-base=/wd/hub &
while ! curl localhost:9515; do echo -n .; sleep 1; done
nightwatch
```
## Chrome Headless
Recently Chrome got native support for running headless, controlled via a remote debugging API. Apparently this could further simplify things, but I couldn’t figure out in reasonable time what exactly it would replace. I suspected it would mean that chrome natively talks webdriver, but I can’t find anything about that in the sparse documentation and this issues sound like.
Which brings me to my initial observation: Figuring out how a thing works is hard, figuring out how things work *together* is so much harder. Let’s all keep that in mind.
