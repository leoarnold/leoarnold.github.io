---
layout: post
title:  "How to use data-test-id in Rails feature tests"
date:   2023-04-23 23:59:41 +0200
tags:
  - capybara
  - rails
---

[`data-*`][custom_data_attributes]

```html
<table></table>
```

```erb
<table>
  <tbody>
    <% collection.each do |thing| %>
      <tr>
        <td><%= thing.name %></td>
      </tr>
    <% end %>
  </tbody>
</table>
```

```ruby
# spec/support/capybara.rb

module Capybara
  module Node
    class Element
      def test_id
        self["data-test-id"]
      end
    end
  end
end
```

```ruby
# spec/support/site_prism.rb

module SitePrism
  class Section
    def test_id
      root_element.test_id
    end
  end
end
```

[custom_data_attributes]: https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/data-*
[uuid]: https://en.wikipedia.org/wiki/Universally_unique_identifier
[to_param]: https://api.rubyonrails.org/classes/ActiveRecord/Integration.html#method-i-to_param
