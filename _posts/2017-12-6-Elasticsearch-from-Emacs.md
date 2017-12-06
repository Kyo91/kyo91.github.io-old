---
title: Elasticsearch Requests from Emacs
---

# Elasticsearch Requests from Emacs

Lately, I've found myself working with Elasticsearch's JSON API and being disappointed with the state of api request tools. While many offer some neat functionalities, none of them are as easy to navigate as Emacs. Additionally, all I've seen require typing into different text fields for say the URL resource versus the body text (on that note many clients seem to have limited support for bodies on a GET request, albeit I've never seen an API besides ES's to use that feature). The battle-tested curl works well in that I'm able to specify everything on the command line and provide body text, however I found editing data on the cli to be a chore and having to type the whole url was getting old when that part never changed. Elasticsearch's documentation shows the request made on the index/type in question with the body right below it.

![Elasticsearch documentation for a create index PUT request](/assets/elastic-create.png)

From this documentation view, elasticsearch allows you to copy the request to curl or run it through kibana. However, I'm not using Kibana and have already stated my issues with using curl directly. Seeing as how I had been writing my bodys in Emacs' `json-mode` for automatic pretty-print indentation, I decided to make the documentation syntax directly runnable in Emacs. After a bit of looking around at Emacs http libraries, I ran into issues with libraries either not documenting/supporting a GET request with a body or libraries that were designed to take in Elisp data structures and interpret them as json (which is great for general elisp programming I'm sure, but the opposite of what I needed here). After a bit of searching, I decided that the easiest way would be to simply send my buffer minus the first line directly to curl itself, while using the first line to setup up request type and resource location. The end result is below:


```emacs-lisp
(defun kyo/es-request ()
  (interactive)
  (goto-char 1)
  (let* ((firstline (thing-at-point 'line t))
         (command-parts (split-string (s-chomp firstline)))
         (json-start (progn (forward-line 1) (point)))
         (body (buffer-substring-no-properties json-start (point-max)))
         (request (concat "curl -X "
                          (first command-parts)
                          " -s http://localhost:9200"
                          (second command-parts)
                          " -d '"
                          body
                          "'"))
         (response-buffer (switch-to-buffer-other-window "*ES_Response*")))
    (json-mode)
    (message request)
    (shell-command request response-buffer)
    (json-mode-beautify)
    (other-window -1)))
```
## Results

With the above function, I can take a buffer like
```json
PUT test
{
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "properties" : {
                "field1" : { "type" : "text" }
            }
        }
    }
}
```

which will run

```bash
curl -X PUT -s http://localhost:9200test -d ’{
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "properties" : {
                "field1" : { "type" : "text" }
            }
        }
    }
}’
```

and return the Elasticsearch result, pretty printed in another buffer.

## Future Work

While I'm very happy with my current workflow here and have taken to using Emacs buffers as my sole interface to elasticsearch (even in case of empty bodies), there are a couple of pain points I hope to address given time in the future. `json-mode` isn't happy with my non-JSON header and I should ideally create a new major mode to better represent what I'm doing. On that note, I've been recording my queries and responses in org-babel source blocks and would like to automate this through usage of babel's code evaluation features.
