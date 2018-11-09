.. index::
    single: Service Container; Decoration

How to Decorate Services
========================

When overriding an existing definition, the original service is lost:

.. configuration-block::

    .. code-block:: yaml

        services:
            app.mailer:
                class: AppBundle\Mailer

            # this replaces the old app.mailer definition with the new one, the
            # old definition is lost
            app.mailer:
                class: AppBundle\NewMailer

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema-instance"
            xsd:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="app.mailer" class="AppBundle\Mailer" />

                <!-- this replaces the old app.mailer definition with the new
                     one, the old definition is lost -->
                <service id="app.mailer" class="AppBundle\NewMailer" />
            </services>
        </container>

    .. code-block:: php

        use AppBundle\Mailer;
        use AppBundle\NewMailer;

        $container->register('app.mailer', Mailer::class);

        // this replaces the old app.mailer definition with the new one, the
        // old definition is lost
        $container->register('app.mailer', NewMailer::class);

Most of the time, that's exactly what you want to do. But sometimes,
you might want to decorate the old one instead (i.e. apply the `Decorator pattern`_).
In this case, the old service should be kept around to be able to reference
it in the new one. This configuration replaces ``app.mailer`` with a new one,
but keeps a reference of the old one  as ``app.decorating_mailer.inner``:

.. configuration-block::

    .. code-block:: yaml

        services:
            # ...

            app.decorating_mailer:
                class:     AppBundle\DecoratingMailer
                decorates: app.mailer
                arguments: ['@app.decorating_mailer.inner']
                public:    false

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema-instance"
            xsd:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <!-- ... -->

                <service id="app.decorating_mailer"
                    class="AppBundle\DecoratingMailer"
                    decorates="app.mailer"
                    public="false"
                >
                    <argument type="service" id="app.decorating_mailer.inner" />
                </service>

            </services>
        </container>

    .. code-block:: php

        use AppBundle\DecoratingMailer;
        use Symfony\Component\DependencyInjection\Reference;

        // ...
        $container->register('app.decorating_mailer', DecoratingMailer::class)
            ->setDecoratedService('app.mailer')
            ->addArgument(new Reference('app.decorating_mailer.inner'))
            ->setPublic(false)
        ;

Here is what's going on here: the ``decorates`` option tells the container that
the ``app.decorating_mailer`` service replaces the ``app.mailer`` service. By
convention, the old ``app.mailer`` service is renamed to
``app.decorating_mailer.inner``, so you can inject it into your new service.

.. tip::

    Most of the time, the decorator should be declared private, as you will not
    need to retrieve it as ``app.decorating_mailer`` from the container.

    The visibility of the decorated ``app.mailer`` service (which is an alias
    for the new service) will still be the same as the original ``app.mailer``
    visibility.

.. note::

    The generated inner id is based on the id of the decorator service
    (``app.decorating_mailer`` here), not of the decorated service (``app.mailer``
    here). This is mandatory to allow several decorators on the same service
    (they need to have different generated inner ids).

    You can change the inner service name if you want to using the
    ``decoration_inner_name`` option:

    .. configuration-block::

        .. code-block:: yaml

            services:
                app.decorating_mailer:
                    # ...
                    decoration_inner_name: app.decorating_mailer.wooz
                    arguments: ['@app.decorating_mailer.wooz']

        .. code-block:: xml

            <?xml version="1.0" encoding="UTF-8" ?>
            <container xmlns="http://symfony.com/schema/dic/services"
                xmlns:xsd="http://www.w3.org/2001/XMLSchema-instance"
                xsd:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

                <services>
                    <!-- ... -->

                    <service
                        id="app.decorating_mailer"
                        class="AppBundle\DecoratingMailer"
                        decorates="app.mailer"
                        decoration-inner-name="app.decorating_mailer.wooz"
                        public="false"
                    >
                        <argument type="service" id="app.decorating_mailer.wooz" />
                    </service>

                </services>
            </container>

        .. code-block:: php

            use AppBundle\DecoratingMailer;
            use Symfony\Component\DependencyInjection\Reference;

            $container->register('app.decorating_mailer', DecoratingMailer::class)
                ->setDecoratedService('app.mailer', 'app.decorating_mailer.wooz')
                ->addArgument(new Reference('app.decorating_mailer.wooz'))
                // ...
            ;

Decoration Priority
-------------------

.. versionadded:: 2.8
    The ability to define the decoration priority was introduced in Symfony 2.8.
    Prior to Symfony 2.8, the priority depends on the order in
    which definitions are found.

When applying multiple decorators to a service, you can control their order with
the ``decoration_priority`` option. Its value is an integer that defaults to
``0`` and higher priorities mean that decorators will be applied earlier.

.. configuration-block::

    .. code-block:: yaml

        foo:
            class: Foo

        bar:
            class: Bar
            public: false
            decorates: foo
            decoration_priority: 5
            arguments: ['@bar.inner']

        baz:
            class: Baz
            public: false
            decorates: foo
            decoration_priority: 1
            arguments: ['@baz.inner']

    .. code-block:: xml

        <?xml version="1.0" encoding="UTF-8" ?>

        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

            <services>
                <service id="foo" class="Foo" />

                <service id="bar" class="Bar" decorates="foo" decoration-priority="5" public="false">
                    <argument type="service" id="bar.inner" />
                </service>

                <service id="baz" class="Baz" decorates="foo" decoration-priority="1" public="false">
                    <argument type="service" id="baz.inner" />
                </service>
            </services>
        </container>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Reference;

        $container->register('foo', 'Foo')

        $container->register('bar', 'Bar')
            ->addArgument(new Reference('bar.inner'))
            ->setPublic(false)
            ->setDecoratedService('foo', null, 5);

        $container->register('baz', 'Baz')
            ->addArgument(new Reference('baz.inner'))
            ->setPublic(false)
            ->setDecoratedService('foo', null, 1);

The generated code will be the following::

    $this->services['foo'] = new Baz(new Bar(new Foo())));

.. _decorator pattern: https://en.wikipedia.org/wiki/Decorator_pattern
