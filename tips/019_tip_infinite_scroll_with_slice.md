---
layout: default
title: Boost infinite scroll performance with Slice
sitemap:
priority: 0.5
lastmod: 2016-11-12T22:22:00-00:00
---

# Boost performance of pagination with infinite scrolling using Slice

__Tip submitted by [@nkolosnjaji](https://github.com/nkolosnjaji)__
__SliceResponseBodyAdvice by [@blagerweij](https://github.com/blagerweij)__

Pagination with infinite scrolling is using Spring Data Page to retrieve entities from your database.
This will trigger two queries, one to fetch entities and second for `count all` to determine the total items for paging. Infinite scrolling doesn't need information about the total size but only if there is a next page to load. To avoid `count all` query which can be an expensive operation when working with large datasets, use [Slice](http://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Slice.html) instead of Page which will boost performance of infinite scrolling.

We will use a custom HTTP header `X-Has-Next-Page` to send information to front-end infinite-scroll plugin.

  * Define new method in your Entity repository:

```
Slice<YourEntity> findSliceBy(Pageable pageable);
```

  * Add a new class called `SliceResponseBodyAdvice`, this will improve the default Spring-Data rendering for Slice responses by adding an HTTP header, and returning the JSON array directly:

```
@ControllerAdvice
public class SliceResponseBodyAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (body instanceof Slice) {
            Slice slice = (Slice) body;
            response.getHeaders().add("X-Has-Next-Page",String.valueOf(slice.hasNext()));
            return slice.getContent();
        }
        return body;
    }
}
```

  * Modify REST controller to call Slice instead of Page. Note that the controller simply returns a Slice<Object>.

```
@GetMapping("/<YourEntities>")
@Timed
public @ResponseBody Slice<YourEntity> getAllYourEntities(Pageable pageable)
    throws URISyntaxException {
    return repoRepository.findSliceBy(pageable);
}
```

  * Define new view model in `entity.controller.js`

```
vm.hasNextPage = false;
```

  * Extract HTTP header value from response and assign it to view model in

```
function onSuccess(data, headers) {
    vm.hasNextPage = headers('X-Has-Next-Page') === "true";
    ...
}
```

  * Use view model with infinite-scroll plugin in `<your-entities>.html`

```
<tbody infinite-scroll="vm.loadPage(vm.page + 1)" infinite-scroll-disabled="!vm.hasNextPage">
```
