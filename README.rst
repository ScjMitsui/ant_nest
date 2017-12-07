========
AntNest
========

Overview
========

AntNest is a simple, clear and fast Web Crawler framework build on python3.6+,  powered by asyncio.

As a Scrapy user, I think scrapy provide many awesome features what I think AntNest should have too.This is some main
difference:

* Scrapy use callback way to write code while AntNest use coroutines
* Scrapy is stable and widely usage while AntNest is in early development
* AntNest has only 600+ lines core code now, and it works

Features
========

* Things(request, response and item) can though pipelines(in async or not)
* Item and item extractor,  it`s easy to define and extract(by xpath for now) a validated(by field type) item
* Custom "ensure_future" and "as_completed" method provide concurrent limit and collection of completed coroutines
* Default coroutines concurrent limit, reduce memory usage

Install
=======
::

    pip install ant_nest

Usage
=====

Let`s take a look, create book.py first::

    from ant_nest.ant import Ant
    from ant_nest.things import StringField, IntField ItemExtractor
    from ant_nest.pipelines import *

    # define a item structure we want to crawl
    class BookItem(Item):
        name = StringField()
        author = StringField()
        content = StringField()
        origin_url = StringField()
        date = IntField(null=True)  # The filed is optional


    # define our ant
    class BookAnt(Ant):
        # the things(request, response, item) will pass through pipelines in order, pipelines can change or drop them
        item_pipelines = [ItemValidatePipeline(),
                          ItemMysqlInsertPipeline(settings.MYSQL_HOST, settings.MYSQL_PORT, settings.MYSQL_USER,
                                                  settings.MYSQL_PASSWORD, settings.MYSQL_DATABASE, 'book'),
                          ReportPipeline()]
        request_pipelines = [RequestDuplicateFilterPipeline(), RequestUserAgentPipeline(), ReportPipeline()]
        response_pipelines = [ResponseRetryPipeline(), ResponseFilterErrorPipeline(), ReportPipeline()]


        # define ItemExtractor to extract item field by xpath from response(html source code)
        self.item_extractor = ItemExtractor(BookItem)
        self.item_extractor.add_xpath('name', '//div/h1/text()')
        self.item_extractor.add_xpath('author', '/html/body/div[1]/div[@class="author"]/text()')
        self.item_extractor.add_xpath('content', '/html/body/div[2]/div[2]/div[2]//text()',
                                      ItemExtractor.join_all)

        # crawl book information
        async def crawl_book(self, url):
            # send request and wait for response
            response = await self.request(url)
            # extract item from response
            item = self.item_extractor.extract(response)
            item.origin_url = str(response.url)  # or item['origin_url'] = str(response.url)
            # wait "collect" coroutine, it will let item pass through "item_pipelines"
            await self.collect(item)

        # app entrance
        async def run(self):
            response = await self.request('https://fake_bookstore.com')
            # extract all book links by xpath ("html_element" is a HtmlElement object from lxml)
            urls = response.html_element.xpath('//a[@class="single_book"]/@href')
            # run "crawl_book" coroutines in concurrent
            for url in urls:
                # "self.ensure_future" is a method like "ensure_future" in "asyncio", but it provide something else
                self.ensure_future(self.crawl_book(url))

Create a settings.py::

    import logging


    logging.basicConfig(level=logging.DEBUG)
    ANT_PACKAGES = ['book']

Then in a console::

    $ant_nest -a book.BookAnt

Defect
======

* Complex exception handle

one coroutine`s exception will break await chain especially in a loop unless we handle it by
hand. eg::

    for cor in self.as_completed((self.crawl(url) for url in self.urls)):
        try:
            await cor
        except Exception:  # may raise many exception in a await chain
            pass

* High memory usage

It`s a "feature" that asyncio eat large memory especially with high concurrent IO, one simple solution is set a
concurrent limit, but it`s complex to get the balance between performance and limit.

Todo
====

* Memory leaks
* Regular expressions extractor support
* Jpath(json-path) extractor support
* Redis pipeline