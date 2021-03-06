[![Build status](https://ci.appveyor.com/api/projects/status/t41dfuti5xtlwlqv)](https://ci.appveyor.com/project/chriseldredge/aspnet-webapi-htmlmicrodataformatter)

This library enhances your Asp.net WebApi project by adding a MediaTypeFormatter
that formats arbitrary objects as well-formed html5 documents with embedded
microdata attributes for entities, properties and values.

The library also includes a controller that generates documentation,
forms and links based on your routes, parameters, controllers and actions.
Enabling this controller to respond to requests, e.g. to `~/api`,
greatly improves how easily clients can discover available resources and
actions and the parameters they accept.

Using this library enables clients that follow HATEOAS (Hypermedia as the Engine of Application State)
conventions that prevent tight coupling, magic URLs and brittle formatting.

## Example

Given this code:

```c#
namespace SampleNamespace
{
    public class TodoController : ApiController
    {
        public Todo Get(int id)
        {
            return new Todo
                {
                    Name = "Finish this app",
                    Description = "It'll take 6 to 8 weeks.",
                    Due = DateTime.Now.AddDays(7*6)
                };
        }
    }

    public class Todo
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public DateTime Due { get; set; }
    }
}
```

The following markup is rendered:

```html
<html>
    <body>
        <dl itemtype="http://example.com/api/doc/SampleNamespace.Todo" itemscope="itemscope">
            <dt>Name</dt>
            <dd><span itemprop="name">Finish this app</span></dd>
            <dt>Description</dt>
            <dd><span itemprop="description">It'll take 6 to 8 weeks.</span></dd>
            <dt>Due</dt>
            <dd><time datetime="2013-09-04T12:59:31Z" itemprop="due">Wed, 04 Sep 2013 12:59:31 GMT</time></dd>
        </dl>
    </body>
</html>
```

When DocumentationController is configured, the following markup is rendered:

```html
<section class="api-group" id="Todo">
    <header><h1>Todo</h1></header>
    <section class="api">
        <h2>Get</h2>
        <p>GET /api/todo</p>
        <form action="/api/todo" method="GET" data-templated="false">
            <label>
                id
                <input name="id" type="text" value="" data-required="true" data-calling-convention="query-string">
            </label>
            <input type="submit">
        </form>
    </section>
</section>
```

## Available on NuGet Gallery

To install the [AspNet.WebApi.HtmlMicrodataFormatter package](http://nuget.org/packages/AspNet.WebApi.HtmlMicrodataFormatter),
run the following command in the [Package Manager Console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)

    PM> Install-Package AspNet.WebApi.HtmlMicrodataFormatter.WebActivator

The above package depends on WebActivator and will drop a c# file in your App_Start folder
and a javascript file in your Scripts folder.

If you do not want to use WebActivator you can install just the library:

    PM> Install-Package AspNet.WebApi.HtmlMicrodataFormatter

## Configuring HtmlMicrodataFormatter

If you are not using the WebActivator package, see
[HtmlMicrodataFormatterActivator.cs](source/AspNet.WebApi.HtmlMicrodataFormatter.WebActivator/HtmlMicrodataFormatterActivator.cs.pp)
for examples of how to configure your project.

## Links

The driving force behind REST and HATEOAS is using links (and forms) as the only means of
transitioning from one state to another. Thus when a response comes from submitting a form
or following a link, the response should in turn contain its own links to related
resources and functionality.

AspNet.WebApi.HtmlMicrodataFormatter.Link can be placed on your response objects to create links
that include rel attributes.

## Uri Templates

The forms generated by DocumentationController may have URIs that contain template expressions,
e.g. `/api/records/{id}`. If the action has one or more template expressions, the form will
have the attribute `data-templated="true"`. Clients should use an [RFC6570](http://tools.ietf.org/html/rfc6570)
template processor on these URIs before submitting them.

A javascript file from [formtemplate-js](https://github.com/themotleyfool/formtemplate-js)
is included in the WebActivator package that will use [uritemplate-js](https://github.com/fxa/uritemplate-js)
to parse and expand template actions when the form is being submitted.

## Customizing Serialization

Serialization is delegated to IHtmlMicrodataSerializer instances which
are registered by type. The first IHtmlMicrodataSerializer that handles
a type or ancestor type is chosen.

The following serializers are automatically registered:

 * UriSerializer: renders System.Uri as `<a>` elements
 * LinkSerializer: renders `AspNet.WebApi.HtmlMicrodataFormatter.Link` as `<a>` with `rel=` attribute
 * DateTimeSerializer: renders System.DateTime and System.DateTimeOffset as `<time>` elements
 * TimeSpanSerializer: renders System.TimeSpan as `<time>` elements using ISO8601 format
 * ToStringSerializer: renders objects as `<span>` elements by invoking ToString() on them
 * ApiDocumentationSerializer: renders SimpleApiDocumentation (used with DocumentationController)
 * ApiDescriptionSerializer: renders SimpleApiDescription as `<form>` (or `<a>` for parameterless actions)
 * TypeDocumentationSerializer: handles TypeDocumentationSerializer used by DocumentationController.GetTypeDocumentation

If none of the registered serializers supports a given type, DefaultSerializer will be used.

By default, ToStringSerializer does not handle any types. You can add types as follows:

```c#
var htmlMicrodataFormatter = new HtmlMicrodataFormatter();
htmlMicrodataFormatter.ToStringSerializer.AddSupportedTypes(typeof(Version));
```

You can register your own serializers by calling `htmlMicrodataFormatter.RegisterSerializer()`.

You can choose to implement IHtmlMicrodataSerializer directly or subclass from either
DefaultSerializer or EntitySerializer.

EntitySerializer allows you to customize individual properties while letting
other properties use default behavior:

```c#
public class Todo
{
    public string Name { get; set; }
    public Uri Icon { get; set; }
}

public class TodoSerializer : EntitySerializer<Todo>
{
    public TodoSerializer()
    {
        Property(todo => todo.Icon, RenderIcon);
    }

    private IEnumerable<XObject> RenderIcon(string propertyName, Uri uri, SerializationContext context)
    {
        if (uri == null) return Enumerable.Empty<XObject>();

        var img = new XElement("img",
            new XAttribute("src", uri.GetComponents(UriComponents.AbsoluteUri, UriFormat.UriEscaped)),
            new XAttribute("alt", "Icon"));

        SetPropertyName(img, propertyName, context);

        return new[] {img};
    }
}
```

## Customizing itemprop

You can control how property names are converted into itemprop attribute values
by setting PropNameProvider on HtmlMicrodataFormatter. The default is to
convert them to camel case.

## TODO

- Mechanism to control order properties are rendered in
- Add more custom input types (checkbox, textarea, select, file, etc)
- Transform `<param/>` and `<returns/>` elements
- Microdata compatible schema from type documentation
