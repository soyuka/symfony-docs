Object Mapping
==============

Symfony provides a mapper to transform a given object to another one.

.. _activating_the_serializer:

Installation
------------

Run this command to install the ``object-mapper`` before using it:

.. code-block:: terminal

    $ composer require symfony/object-mapper

Using the ObjectMapper Service
------------------------------

Once enabled, the object mapper service can be injected in any service where
you need it or it can be used in a controller::

    // src/Controller/DefaultController.php
    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;

    class DefaultController extends AbstractController
    {
        public function index(ObjectMapperInterface $objectMapper): Response
        {
            // keep reading for usage examples
        }
    }


Map an object to another one
----------------------------

To map an object to another one use ``map``::

    use App\Dto\Source;
    use App\Dto\Target;

    $mapper = new ObjectMapper();
    $mapper->map(source: new Source(), target: Target::class);


If you alread have a target object, you can use its instance directly::

    use App\Dto\Source;
    use App\Dto\Target;

    $target = new Target();
    $mapper = new ObjectMapper();
    $mapper->map(source: new Source(), target: $target);


Configure the mapping target using attributes
---------------------------------------------

The Object Mapper component includes a :class:`Symfony\\Component\\ObjectMapper\\Attribute\\Map` attribute to configure mapping
behavior between objects. Use this attribute on a class to specify the
target class::

    // src/Dto/Source.php
    namespace App\Dto;

    use Symfony\Component\ObjectMapper\Attributes\Map;

    #[Map(target: Target::class)]
    class Source {}

Configure property mapping
--------------------------

Use the :class:`Symfony\\Component\\ObjectMapper\\Attribute\\Map` attribute on properties to configure property mapping between
objects. ``target`` changes the target property, ``if`` allows to
conditionnally map properties::

    // src/Dto/Source.php
    namespace App\Dto;

    use Symfony\Component\ObjectMapper\Attributes\Map;

    class Source {
        #[Map(target: 'fullName')]
        public string $firstName;

        #[Map(if: false)]
        public string $lastName;
    }

Transform mapped values
-----------------------

Use ``transform`` to call a function or a
:class:`Symfony\Component\ObjectMapper\CallableInterface`::

    // src/ObjectMapper/TransformNameCallable
    namespace App\ObjectMapper;

    use App\Dto\Source;
    use Symfony\Component\ObjectMapper\CallableInterface;

    /**
     * @implements CallableInterface<Source>
     */
    final class TransformNameCallable implements CallableInterface
    {
        public function __invoke(mixed $value, object $object): mixed
        {
            return sprintf('%s %s', $object->firstName, $object->lastName);
        }
    }

    // src/Dto/Source.php
    namespace App\Dto;

    use App\ObjectMapper\TransformNameCallable;
    use Symfony\Component\ObjectMapper\Attributes\Map;

    class Source {
        #[Map(target: 'fullName', transform: TransformNameCallable::class)]
        public string $firstName;

        #[Map(if: false)]
        public string $lastName;
    }


The ``if`` and ``transform`` parameters also accept callbacks::

    // src/Dto/Source.php
    namespace App\Dto;

    use App\ObjectMapper\TransformNameCallable;
    use Symfony\Component\ObjectMapper\Attributes\Map;

    class Source {
        #[Map(if: 'boolval', transform: 'ucfirst')]
        public ?string $lastName = null;
    }

The :class:`Symfony\\Component\\ObjectMapper\\Attribute\\Map` attribute works on
classes and it can be repeated::

    // src/Dto/Source.php
    namespace App\Dto;

    use App\Dto\B;
    use App\Dto\C;
    use App\ObjectMapper\TransformNameCallable;
    use Symfony\Component\ObjectMapper\Attributes\Map;

    #[Map(target: B::class, if: [Source::class, 'shouldMapToB'])]
    #[Map(target: C::class, if: [Source::class, 'shouldMapToC'])]
    class Source {
        public static function shouldMapToB(mixed $value, object $object): bool
        {
            return false;
        }

        public static function shouldMapToC(mixed $value, object $object): bool
        {
            return true;
        }
    }

Provide your own mapping metadata
---------------------------------

The :class:`Symfony\\Component\\ObjectMapper\\MapperMetadataFactoryInterface` allows
to change how mapping metadata is computed. With this interface we can create a
`MapStruct`_ version of the Object Mapper::

    // src/ObjectMapper/Metadata/MapStructMapperMetadataFactory.php
    namespace App\Metadata\ObjectMapper;

    use Symfony\Component\ObjectMapper\Attribute\Map;
    use Symfony\Component\ObjectMapper\Metadata\MapperMetadataFactoryInterface;
    use Symfony\Component\ObjectMapper\Metadata\Mapping;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;

    /**
     * A Metadata factory that implements the basics behind https://mapstruct.org/.
     */
    final class MapStructMapperMetadataFactory implements MapperMetadataFactoryInterface
    {
        public function __construct(private readonly string $mapper)
        {
            if (!is_a($mapper, ObjectMapperInterface::class, true)) {
                throw new \RuntimeException(sprintf('Mapper should implement "%s".', ObjectMapperInterface::class));
            }
        }

        public function create(object $object, ?string $property = null, array $context = []): array
        {
            $refl = new \ReflectionClass($this->mapper);
            $mapTo = [];
            $source = $property ?? $object::class;
            foreach (($property ? $refl->getMethod('map') : $refl)->getAttributes(Map::class) as $mappingAttribute) {
                $map = $mappingAttribute->newInstance();
                if ($map->source === $source) {
                    $mapTo[] = new Mapping(source: $map->source, target: $map->target, if: $map->if, transform: $map->transform);

                    continue;
                }
            }

            // Default is to map properties to a property of the same name
            if (!$mapTo && $property) {
                $mapTo[] = new Mapping(source: $property, target: $property);
            }

            return $mapTo;
        }
    }

With this metadata usage, the mapping definition belongs to a mapper class::

    // src/ObjectMapper/AToBMapper

    namespace App\Metadata\ObjectMapper;

    use App\Dto\Source;
    use App\Dto\Target;
    use Symfony\Component\ObjectMapper\Attributes\Map;
    use Symfony\Component\ObjectMapper\ObjectMapper;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;


    #[Map(source: Source::class, target: Target::class)]
    class AToBMapper implements ObjectMapperInterface
    {
        public function __construct(private readonly ObjectMapper $objectMapper)
        {
        }

        #[Map(source: 'propertyA', target: 'propertyD')]
        #[Map(source: 'propertyB', if: false)]
        public function map(object $source, object|string|null $target = null): object
        {
            return $this->objectMapper->map($source, $target);
        }
    }


The custom metadata is injected inside our :class:`Symfony\\Component\\ObjectMapper\\ObjectMapperInterface`::

    $a = new Source('a', 'b', 'c');
    $metadata = new MapStructMapperMetadataFactory(AToBMapper::class);
    $mapper = new ObjectMapper($metadata);
    $aToBMapper = new AToBMapper($mapper);
    $b = $aToBMapper->map($a);


.. _`MapStruct`: https://mapstruct.org/
