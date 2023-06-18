---
layout: post
title: Delegate functionality with Object-backed View Components
date: 2023-06-18 16:00:00 +0000
tags: ruby
---

Rendering server-side UI components in Ruby on Rails can be made easier with an open-source gem called [View Component](https://viewcomponent.org).
Here is the description of View Component, from the documentation.

> A framework for creating reusable, testable & encapsulated view components, built to integrate seamlessly with Ruby on Rails.

A component is composed of a component Ruby class, component markup (in `.html.erb`), and a call to render the component in a view. Below is a sample implementation from the documentation.

_Component class_
```ruby
# app/components/message_component.rb
class MessageComponent < ViewComponent::Base
  def initialize(name:)
    @name = name
  end
end
```

_Component markup_
```erb
<%# app/components/message_component.html.erb %>
<h1>Hello, <%= @name %>!</h1>
```
```erb
<%# app/views/demo/index.html.erb %>
<%= render(MessageComponent.new(name: "World")) %>
```

Which produces the following markup.
```html
<h1>Hello, World!</h1>
```

## Conventional usage of View Components

In the Ruby on Rails codebases that I've worked on, I've had the opportunity View Components extensively. There are two typical use cases of View Components that I've seen. First is a more generic component which is primarily defined by the attributes passed into it. This could include buttons, pagination components, or links.

Another variant of a View Component is defined primarily by an object that might have some relationship to the domain and is more complex. This could be something like an event or an about me card. These typically have several related attributes and may likely already be represented by a domain object.

Although View Components have many benefits, there are sometimes cases where following the conventional approach becomes unwieldy, especially in cases like the second, where there is a component that is representing an object with several related attributes. Here is a contrived example of an event component, where the purpose of the component is to represent an event that can be attended.

_Component class_
```ruby
# app/components/event_component.rb
class EventComponent < ViewComponent::Base
  def initialize(name:, date:, address:, spots_left:, ticket_cost:, organizer_email:)
    @name = name
    @date = date
    @address = address
    @spots_left = spots_left
    @ticket_cost = ticket_cost
    @organizer_email = organizer_email
  end
end
```

_Component markup_
```erb
<%# app/components/event_component.html.erb %>
<div>
  <h1><%= @name %></h1>
  <p><%= @date %></p>
  <p><%= @address %></p>
  <p><%= @spots_left %></p>
  <p><%= @ticket_cost %></p>
  <p><%= @organizer_email %></p>
</div>
```

_Instantiated and passed to `render`_
```erb
<%# app/views/demo/index.html.erb %>
<%=
  render(
    EventComponent.new(
      name: "Sydney.rb",
      date: 2.weeks.from_now,
      address: "P. Sherman 42 Wallaby Way",
      spots_left: 12,
      ticket_cost: "$42",
      organizer_email: "pixar@nemo.com"
    )
  )
%>
```

Even though this is a contrived example, in reality, I've seen cases where the attributes or methods of an object were being accessed and called to populate the attributes of a related View Component.

```erb
<%# app/views/demo/index.html.erb %>
<%=
  render(
    EventComponent.new(
      name: event.name,
      date: event.date,
      address: event.address,
      spots_left: event.spots_left,
      ticket_cost: event.ticket_cost,
      organizer_email: event.organizer_email
    )
  )
%>
```

Unfortunately, this can introduce the question: "Where does the presentational code live"? In the object? In the View Component? In the markdown? In a module? Inevitably some of the data in the object needs to be capitalized or turned into currency, or pretty printed, etc. This can lead to code in View Components like below.

At face value, this doesn't seem like much of an issue, but it can lead to awkward separation of concerns, strange method naming, and annoying testing, among other things. The View Component should be concerned about rendering a component, not about the presentational concerns of the object it is representing.

As an alternative to this frustration, consider using Object-backed View Components!

## The pattern

When you have a UI component that is conceptually related to a Ruby object, instead of passing in several attributes to the View Component constructor, consider passing in the entire object, and delegating method calls using ActiveSupport `delegate` ([source](https://www.rubydoc.info/gems/activesupport/Module:delegate)).

Following the example from before, suppose you have an event you need to render. Instead of putting the presentational logic in the View Component, instead keep it in the object (as an alternative, you could use the decorator pattern).

```ruby
# app/models/event.rb
class Event
  attr_reader :name, :date, :address, :organizer_email

  def initialize(name:, date:, address:, organizer_email:)
    @name = name
    @date = date
    @address = address
    @organizer_email = organizer_email
  end

  def human_readable_date
    # presentational logic
  end

  def spots_left
    # some api call
  end

  def ticket_cost
    # some api call
  end

  # ... some presentational methods
end
```

Instead of passing in each of the event attributes, instead pass in the event instance.

```erb
<%# app/views/demo/index.html.erb %>
<%= render(EventComponent.new(event: event)) %>
```

Then in the View Component class delegate method calls to the `event` object.

```ruby
# app/components/event_component.rb
class EventComponent < ViewComponent::Base
  delegate :name,
           :date,
           :address,
           :organizer_email,
           :human_readable_date,
           :spots_left,
           :ticket_cost
           to: :@event

  def initialize(event:)
    @event = event
  end
end
```

Then in the view code, call methods like so.

```erb
<%# app/components/event_component.html.erb %>
<div>
  <h1><%= name %></h1>
  <p><%= date %></p>
  <p><%= address %></p>
  <p><%= spots_left %></p>
  <p><%= ticket_cost %></p>
  <p><%= organizer_email %></p>
</div>
```

Now you have a clearer separation of concerns, unit testing the component will be much easier (since now you can use a dummy object in your component test), and it is easier to organize your presentational code that relates to the object you're rendering.

## Conclusion

Carefully consider when is appropriate to use this pattern. It kind of goes against some of the standard approaches demonstrated in the View Component documentation. Do you need a custom component that is conceptually coupled to a domain object? Consider using it. Do you need a generic component with a small number of attributes? Eh, it's probably overkill.

Thanks for reading. Please reach out to me at [zanebliss@icloud.com](mailto:zanebliss@icloud.com) if you have questions or comments! I'd love to chat.
