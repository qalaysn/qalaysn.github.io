---
title:  "OSINT tricks. Working with Binary Edge API"
date:   2024-08-31 05:00:00 +0300
header:
  teaser: "/assets/images/2/osint.png"
categories: 
  - tutorial
tags:
  - osint
  - intelligence
  - apt
  - hacking
---

This post was prepared by me for an Australian HVCK magazine, and I thought it would be useful for OSINT researchers and would be a good start for our OSINT blog.     

![welcome](/assets/images/1/osint.png){:class="img-responsive"}      

Also maybe worth making changes to the 2023 script as there were some changes in their API.     

This blog was started by my colleague and a nice guy - [Ayan](https://www.linkedin.com/in/ayansatybaldinov/), please, support his channel in Telegram [https://t.me/osinteka]([https://t.me/osinteka]), It is for Russian-speaking audience for now, but there will be some interesting case studies, I promise you!         

Perhaps you have encountered [Shodan](https://www.shodan.io/) on numerous occasions:     

![shodan](/assets/images/2/2023-03-05_11-50.png){:class="img-responsive"}          

There is also another amazing resource - [BinaryEdge](https://binaryedge.io)       

Registration on the site is free. 

![login](/assets/images/2/2023-03-05_01-44.png){:class="img-responsive"}          

After logging in, you can request an API token:    

![token](/assets/images/2/2023-03-05_01-47.png){:class="img-responsive"}          

It has great [documentation](https://app.binaryedge.io/account/api):        

![token](/assets/images/2/2023-03-05_01-45.png){:class="img-responsive"}          

> API V2 - For Free, Starter and Business accounts.

### practical example

Let's create a simple program, that will access the BinaryEdge API and search for open elasticsearch databases.     
First of all, create simple function for request:     

```python

# your binaryedge.io API key
BE_API_KEY = '6ec989f3-b13e-4ae7-9836-4294fbdec957'

def binary_edge_request(query, page):
    headers = {'X-Key' : BE_API_KEY}
    url = 'https://api.binaryedge.io/v2/query/search?query=' + query + '&page=' + str(page)
    req = requests.get(url, headers=headers)
    req_json = json.loads(req.content)
    try:
        if req_json.get("status"):
            print (Colors.YELLOW + req_json.get("status") + ' ' + req_json.get("message") + Colors.ENDC)
        else:
            print (Colors.GREEN + "total results: " + str(req_json.get('total')) + Colors.ENDC)
    except Exception as e:
        print (Colors.RED + f"error {e} :(" + bcolors.ENDC)
        sys.exit()
    return req_json.get("events")
```

In this function `/v2/query/search` - can be used with certain parameters and/or full text search.     

For simplicity, we just print `http://<ip>:<port>/_cat/indices` link to elasticsearch indexes list:     

```python
def run(query):
    elastic_q = f'"elasticsearch" {query}'.strip()
    elastic_q = urllib.parse.quote(elastic_q)
    elastic_q = f'type:{elastic_q}'
    print (Colors.YELLOW + f'{elastic_q}....' + Colors.ENDC)
    for page in range(1, 2):
        results = binary_edge_request(elastic_q, page)
        if results:
            for service in results:
                print(Colors.BLUE + f"http://{service['target']['ip']}:{service['target']['port']}/_cat/indices" + Colors.ENDC)
```

Also, for simplicity, we just only get the first page of results:     

```python
for page in range(1, 2):
```

So, the full source code of our script is looks like this (`search.py`):    


```python
import argparse
import json
import requests
import urllib.parse

class Colors:
    HEADER = '\033[95m'
    BLUE = '\033[94m'
    GREEN = '\033[92m'
    YELLOW = '\033[93m'
    RED = '\033[91m'
    PURPLE = '\033[95m'
    ENDC = '\033[0m'

# your binaryedge.io API key
BE_API_KEY = '6ec989f3-b13e-4ae7-9836-4294fbdec957'

# format docs count
def millify(n):
    millnames = ['',' k',' mln',' bln',' trln']
    n = float(n)
    millidx = max(0,min(len(millnames)-1,
                        int(math.floor(0 if n == 0 else math.log10(abs(n))/3))))

    return '{:.0f}{}'.format(n / 10**(3 * millidx), millnames[millidx])

# format index size human readable
def sizeof_fmt(num, suffix='B'):
    for unit in ['','K','M','G','T','P','E','Z']:
        if abs(num) < 1024.0:
            return "%3.1f%s%s" % (num, unit, suffix)
        num /= 1024.0
    return "%.1f%s%s" % (num, 'Y', suffix)

def binary_edge_request(query, page):
    headers = {'X-Key' : BE_API_KEY}
    url = 'https://api.binaryedge.io/v2/query/search?query=' + query + '&page=' + str(page)
    req = requests.get(url, headers=headers)
    req_json = json.loads(req.content)
    try:
        if req_json.get("status"):
            print (Colors.YELLOW + req_json.get("status") + ' ' + req_json.get("message") + Colors.ENDC)
        else:
            print (Colors.GREEN + "total results: " + str(req_json.get('total')) + Colors.ENDC)
    except Exception as e:
        print (Colors.RED + f"error {e} :(" + bcolors.ENDC)
        sys.exit()
    return req_json.get("events")

def run(query):
    elastic_q = f'"elasticsearch" {query}'.strip()
    elastic_q = urllib.parse.quote(elastic_q)
    elastic_q = f'type:{elastic_q}'
    print (Colors.YELLOW + f'{elastic_q}....' + Colors.ENDC)
    for page in range(1, 2):
        results = binary_edge_request(elastic_q, page)
        if results:
            for service in results:
                print(Colors.BLUE + f"http://{service['target']['ip']}:{service['target']['port']}/_cat/indices" + Colors.ENDC)

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='binary edge search')
    parser.add_argument('-q', '--query', required=True, help='binary edge search filter')
    parser.add_argument('-f', '--from', required = False, help = "start page", default = 1)
    parser.add_argument('-t', '--to', required = False, help = "finish page", default = 2)
    args = parser.parse_args()
    query = args.query
    run(query)
```

Run it:    

```bash
python3 search.py -q "users"
```

![result](/assets/images/2/2023-03-05_12-34.png){:class="img-responsive"}     

As you can see, everything is worked perfectly! =^..^=     

Run with another keyword `meow`:    

```bash
python3 search.py -q "meow"
```

![meow](/assets/images/2/2023-03-05_17-08.png){:class="img-responsive"}     

Hundreds of publicly accessible, unprotected databases are the subject of an automated `meow` attack that destroys data without explanation. This behavior began in 2020 by targeting Elasticsearch and MongoDB instances without leaving a notice or ransom demand. Attacks then expanded to other database types and to file systems available on the web.      

Also, in practice, you can make sure that the neglect of database protection can lead to leakage of personal data or banking secrecy:     

![bank](/assets/images/2/2023-03-05_17-36.png){:class="img-responsive"}    

I hope that this simple example will serve as a basis for your programs with more interesting logic.     

I hope this post if useful for entry level cybersec and OSINT specialists and also for professionals.     

By [@cocomelonc](https://www.linkedin.com/in/zhassulan-zhussupov-5a347419b/) for OSINTeka Research Lab.               

### References

[BinaryEdge API](https://docs.binaryedge.io/api-v2/)     
[BinaryEdge pricing](https://www.binaryedge.io/pricing.html)    

Thanks for your time happy hacking and good bye!   
*PS. All drawings and screenshots are mine*    