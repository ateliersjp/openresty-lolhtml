# OpenResty meets LOL HTML
The Docker image is based on [`openresty/openresty:bionic`](https://hub.docker.com/r/openresty/openresty) because `bionic` was the last version of Ubuntu to support `cssparser`. The [Lua binding](https://github.com/jdesgats/lua-lolhtml) for [`lol-html`](https://github.com/cloudflare/lol-html) and its dependencies are pre-installed and ready to rewrite HTML contents.

## Usage
When you `pull` or `run` the Docker image, just use `ateliersjp/openresty-lolhtml:bionic` instead of `openresty/openresty:bionic` and then ready to use `lua-lolhtml`.

## Example
Rewrite insecure hyperlinks:
```
header_filter_by_lua_block {
    if ngx.header.content_type:find("text/html", 1, true) then
        ngx.header.content_length = nil

        local lolhtml = require "lolhtml"
        ngx.ctx.html_rewriter = coroutine.wrap(function(chunk)
            local buffered = ""
            local builder = lolhtml.new_rewriter_builder()

            builder:add_element_content_handlers({
                selector = lolhtml.new_selector("a[href]"),
                element_handler = function(el)
                    local href = el:get_attribute("href")
                                   :gsub("http:", "https:")
                    el:set_attribute("href", href)
                end
            })

            local rewriter = lolhtml.new_rewriter({
                builder = builder,
                sink = function(output)
                    buffered = buffered..output
                end
            })

            while chunk do
                rewriter:write(chunk)
                chunk, buffered = buffered, ""
                chunk = coroutine.yield(chunk)
            end

            rewriter:close()
            return buffered
        end)
    end
}

body_filter_by_lua_block {
    if ngx.ctx.html_rewriter then
        ngx.arg[1] = ngx.ctx.html_rewriter(ngx.arg[1])
    end
}
```
