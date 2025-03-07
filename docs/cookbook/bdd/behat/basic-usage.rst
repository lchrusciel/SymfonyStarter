Basic Usage
===========

The best way of understanding how things work in detail is showing and analyzing examples, that is why this section gathers all the knowledge from the previous chapters.
Let's assume that we are going to implement the functionality of managing countries in our system.
Now let us show you the flow.

Describing features
-------------------

Let's start with writing our feature file, which will contain answers to the most important questions:
Why (benefit, business value), who (actor using the feature) and what (the feature itself).
It should also include scenarios, which serve as examples of how things supposed to work.
Let's have a look at the ``features/addressing/managing_countries/adding_country.feature`` file.

.. code-block:: gherkin

    # features/addressing/managing_countries/adding_country.feature

    @managing_countries
    Feature: Adding a new country
        In order to sell my goods to different countries
        As an Administrator
        I want to add a new country to the store

        Background:
            Given I am logged in as an administrator

        @ui
        Scenario: Adding country
            When I want to add a new country
            And I choose "United States"
            And I add it
            Then I should be notified that it has been successfully created
            And the country "United States" should appear in the store

Pay attention to the form of these sentences. From the developer point of view they are hiding the details of the feature's implementation.
Instead of describing "When I click on the select box And I choose United States from the dropdown Then I should see the United States country in the table"
- we are using sentences that are less connected with the implementation, but more focused on the effects of our actions.
A side effect of such approach is that it results in steps being really generic, therefore if we want to add another way of testing this feature for instance in the domain or api context,
it will be extremely easy to apply. We just need to add a different tag (in this case "@domain") and of course implement the proper steps in the domain context of our system.
To be more descriptive let's imagine that we want to check if a country is added properly in two ways.
First we are checking if the adding works via frontend, so we are implementing steps that are clicking, opening pages,
filling fields on forms and similar, but also we want to check this action regardlessly of the frontend, for that we need the domain, which allows us to perform actions only on objects.

Choosing a correct suite
------------------------

After we are done with a feature file, we have to create a new suite for it. At the beginning we have decided that it will be a frontend/user interface feature, that is why we are placing it in "src/Behat/Resources/suites/ui/addressing/managing_countries.yml".

.. code-block:: yaml

    # src/Behat/Resources/suites/ui/addressing/managing_countries.yml

    default:
        suites:
            ui_managing_countries:
                contexts:
                    # This service is responsible for clearing database before each scenario,
                    # so that only data from the current and its background is available.
                    - App\Behat\Context\Hook\DoctrineORMContext

                    # The transformer contexts services are responsible for all the transformations of data in steps:
                    # For instance "And the country "France" should appear in the store" transforms "(the country "France")" to a proper Country object, which is from now on available in the scope of the step.
                    - App\Behat\Context\Transform\CountryContext
                    - App\Behat\Context\Transform\SharedStorageContext

                    # The setup contexts here are preparing the background, adding available countries and users or administrators.
                    # These contexts have steps like "I am logged in as an administrator" already implemented.
                    - App\Behat\Context\Setup\GeographicalContext
                    - App\Behat\Context\Setup\SecurityContext

                    # Lights, Camera, Action!
                    # Those contexts are essential here we are placing all action steps like "When I choose "France" and I add it Then I should ne notified that...".
                    - App\Behat\Context\Ui\Backend\ManagingCountriesContext
                    - App\Behat\Context\Ui\Backend\NotificationContext
                filters:
                    tags: "@managing_countries && @ui"

A very important thing that is done here is the configuration of tags, from now on Behat will be searching for all your features tagged with ``@managing_countries`` and your scenarios tagged with ``@ui``.
Second thing is ``contexts_services:`` in this section we will be placing all our services with step implementation.

We have mentioned with the generic steps we can easily switch our testing context to @domain. Have a look how it looks:

.. code-block:: yaml

    # src/Behat/Resources/config/suites/domain/addressing/managing_countries.yml

    default:
        suites:
            domain_managing_countries:
                contexts_services:
                    - App\Behat\Context\Hook\DoctrineORMContext

                    - App\Behat\Context\Transform\CountryContext
                    - App\Behat\Context\Transform\SharedStorageContext

                    - App\Behat\Context\Setup\GeographicalContext
                    - App\Behat\Context\Setup\SecurityContext

                    # Domain step implementation.
                    - App\Behat\Context\Domain\Backend\ManagingCountriesContext
                filters:
                    tags: "@managing_countries && @domain"

We are almost finished with the suite configuration.

Registering Pages
-----------------

The page object approach allows us to hide all the detailed interaction with ui (html, javascript, css) inside.

We have three kinds of pages:
    - Page - First layer of our pages it knows how to interact with DOM objects. It has a method ``getUrl(array $urlParameters)`` where you can define a raw url to open it.
    - SymfonyPage - This page extends the Page. It has a router injected so that the ``getUrl()`` method generates a url from the route name which it gets from the ``getRouteName()`` method.
    - Base Crud Pages (IndexPage, CreatePage, UpdatePage) - These pages extend SymfonyPage and they are specific to the Sylius resources. They have a resource name injected and therefore they know about the route name.

There are two ways to manipulate UI - by using ``getDocument()`` or ``getElement('your_element')``.
First method will return a ``DocumentElement`` which represents an html structure of the currently opened page,
second one is a bit more tricky because it uses the ``->getDefinedElements(): array`` method and it will return a ``NodeElement`` which represents only the restricted html structure.

Usage example of ``getElement('your_element')`` and ``getDefinedElements()`` methods.

.. code-block:: php

    final class CreatePage extends SymfonyPage implements CreatePageInterface
    {
        // This method returns a simple associative array, where the key is the name of your element and the value is its locator.
        protected function getDefinedElements(): array
        {
            return array_merge(parent::getDefinedElements(): array, [
                'provinces' => '#sylius_country_provinces',
            ]);
        }

        // By default it will assume that your locator is css.
        // Example with xpath.
        protected function getDefinedElements(): array
        {
            return array_merge(parent::getDefinedElements(): array, [
                'provinces_css' => '.provinces',
                'provinces_xpath' => ['xpath' => '//*[contains(@class, "provinces")]'], // Now your value is an array where key is your locator type.
            ]);
        }

        // Like that you can easily manipulate your page elements.
        public function addProvince(ProvinceInterface $province): void
        {
            $provinceSelectBox = $this->getElement('provinces');

            $provinceSelectBox->selectOption($province->getName());
        }
    }

Let's get back to our main example and analyze our scenario. We have steps like:

.. code-block:: gherkin

    When I choose "France"
    And I add it
    Then I should be notified that it has been successfully created
    And the country "France" should appear in the store

.. code-block:: php

    namespace App\Behat\Page\Backend\Country;

    use App\Behat\Page\Backend\Crud\CreatePage as BaseCreatePage;

    final class CreatePage extends BaseCreatePage implements CreatePageInterface
    {
        /**
         * @param string $name
         */
        public function chooseName(string $name): void
        {
            $this->getDocument()->selectFieldOption('Name', $name);
        }

        public function create(): void
        {
            $this->getDocument()->pressButton('Create');
        }
    }

.. code-block:: php

    namespace App\Behat\Page\Backend\Country;

    use App\Behat\Page\Backend\Crud\IndexPage as BaseIndexPage;

    final class IndexPage extends BaseIndexPage implements IndexPageInterface
    {
        /**
         * @return bool
         */
        public function isSingleResourceOnPage(array $parameters): bool
        {
            try {
                // Table accessor is a helper service which is responsible for all html table operations.
                $rows = $this->tableAccessor->getRowsWithFields($this->getElement('table'), $parameters);

                return 1 === count($rows);
            } catch (ElementNotFoundException $exception) {
                // Table accessor throws this exception when cannot find table element on page.
                return false;
            }
        }
    }

.. warning::

    There is one small gap in this concept - PageObjects is not a concrete instance of the currently opened page, they only mimic its behaviour (dummy pages).
    This gap will be more understandable on the below code example.

.. code-block:: php

    // Of course this is only to illustrate this gap.

    class HomePage
    {
        // In this context on home page sidebar you have for example weather information in selected countries.
        public function readWeather()
        {
            return $this->getElement('sidebar')->getText();
        }

        protected function getDefinedElements(): array
        {
            return ['sidebar' => ['css' => '.sidebar']]
        }

        protected function getUrl()
        {
            return 'http://your_domain.com';
        }
    }

    class LeagueIndexPage
    {
        // In this context you have for example football match results.
        public function readMatchResults()
        {
            return $this->getElement('sidebar')->getText();
        }

        protected function getDefinedElements(): array
        {
            return ['sidebar' => ['css' => '.sidebar']]
        }

        protected function getUrl()
        {
            return 'http://your_domain.com/leagues/'
        }
    }

    final class GapContext implements Context
    {
        private $homePage;
        private $leagueIndexPage;

        /**
         * @Given I want to be on Homepage
         */
        public function iWantToBeOnHomePage() // After this method call we will be on "http://your_domain.com".
        {
            $this->homePage->open(); //When we add @javascript tag we can actually see this thanks to selenium.
        }

        /**
         * @Then I want to see the sidebar and get information about the weather in France
         */
        public function iWantToReadSideBarOnHomePage($someInformation) // Still "http://your_domain.com".
        {
            $someInformation === $this->leagueIndexPage->readMatchResults() // This returns true, but wait a second we are on home page (dummy pages).

            $someInformation === $this->homePage->readWeather() // This also returns true.
        }
    }

Registering contexts
--------------------

As it was shown in the previous section we have registered a lot of contexts, so we will show you only some of the steps implementation.

.. code-block:: gherkin

    Given I want to add a new country
    And I choose "United States"
    And I add it
    Then I should be notified that it has been successfully created
    And the country "United States" should appear in the store

Let's start with essential one ManagingCountriesContext

Ui contexts
~~~~~~~~~~~

.. code-block:: php

    namespace App\Behat\Context\Ui\Backend

    use Behat\Behat\Context\Context;

    final class ManagingCountriesContext implements Context
    {
        /**
         * @var IndexPageInterface
         */
        private $indexPage;

        /**
         * @var CreatePageInterface
         */
        private $createPage;

        /**
         * @var UpdatePageInterface
         */
        private $updatePage;

        /**
         * @param IndexPageInterface $indexPage
         * @param CreatePageInterface $createPage
         * @param UpdatePageInterface $updatePage
         */
        public function __construct(
            IndexPageInterface $indexPage,
            CreatePageInterface $createPage,
            UpdatePageInterface $updatePage
        ) {
            $this->indexPage = $indexPage;
            $this->createPage = $createPage;
            $this->updatePage = $updatePage;
        }

        /**
         * @Given I want to add a new country
         */
        public function iWantToAddNewCountry(): void
        {
            $this->createPage->open(); // This method will send request.
        }

        /**
         * @When I choose :countryName
         */
        public function iChoose($countryName): void
        {
            $this->createPage->chooseName($countryName);
            // Great benefit of using page objects is that we hide html manipulation behind a interfaces so we can inject different CreatePage which implements CreatePageInterface
            // And have different html elements which allows for example chooseName($countryName).
        }

        /**
         * @When I add it
         */
        public function iAddIt(): void
        {
            $this->createPage->create();
        }

        /**
         * @Then /^the (country "([^"]+)") should appear in the store$/
         */
        public function countryShouldAppearInTheStore(CountryInterface $country): void // This step use Country transformer to get Country object.
        {
            $this->indexPage->open();

            //Webmozart assert library.
            Assert::true(
                $this->indexPage->isSingleResourceOnPage(['code' => $country->getCode()]),
                sprintf('Country %s should exist but it does not', $country->getCode())
            );
        }
    }

.. code-block:: php

    namespace App\Behat\Context\Ui\Backend

    use Behat\Behat\Context\Context;

    final class NotificationContext implements Context
    {
        /**
         * This is a helper service which give access to proper notification elements.
         *
         * @var NotificationCheckerInterface
         */
        private $notificationChecker;

        /**
         * @param NotificationCheckerInterface $notificationChecker
         */
        public function __construct(NotificationCheckerInterface $notificationChecker)
        {
            $this->notificationChecker = $notificationChecker;
        }

        /**
         * @Then I should be notified that it has been successfully created
         */
        public function iShouldBeNotifiedItHasBeenSuccessfullyCreated(): void
        {
            $this->notificationChecker->checkNotification('has been successfully created.', NotificationType::success());
        }
    }

Transformer contexts
~~~~~~~~~~~~~~~~~~~~

.. code-block:: php

    namespace App\Behat\Context\Transform;

    use Behat\Behat\Context\Context;

    final class CountryContext implements Context
    {
        /**
         * @var CountryNameConverterInterface
         */
        private $countryNameConverter;

        /**
         * @var RepositoryInterface
         */
        private $countryRepository;

        /**
         * @param CountryNameConverterInterface $countryNameConverter
         * @param RepositoryInterface $countryRepository
         */
        public function __construct(
            CountryNameConverterInterface $countryNameConverter,
            RepositoryInterface $countryRepository
        ) {
            $this->countryNameConverter = $countryNameConverter;
            $this->countryRepository = $countryRepository;
        }

        /**
         * @Transform /^country "([^"]+)"$/
         * @Transform /^"([^"]+)" country$/
         */
        public function getCountryByName(string $countryName): Country // Thanks to this method we got in our ManagingCountries an Country object.
        {
            $countryCode = $this->countryNameConverter->convertToCode($countryName);
            $country = $this->countryRepository->findOneBy(['code' => $countryCode]);

            Assert::notNull(
                $country,
                'Country with name %s does not exist'
            );

            return $country;
        }
    }


.. code-block:: php

    namespace App\Behat\Context\Ui\Backend;

    use App\Behat\Page\Backend\Country\UpdatePageInterface;
    use Behat\Behat\Context\Context;

    final class ManagingCountriesContext implements Context
    {
        /**
         * @var UpdatePageInterface
         */
        private $updatePage;

        /**
         * @param UpdatePageInterface $updatePage
         */
        public function __construct(UpdatePageInterface $updatePage)
        {
            $this->updatePage = $updatePage;
        }

        /**
         * @Given /^I want to create a new province in (country "[^"]+")$/
         */
        public function iWantToCreateANewProvinceInCountry(CountryInterface $country)
        {
            $this->updatePage->open(['id' => $country->getId()]);

            $this->updatePage->clickAddProvinceButton();
        }
    }

.. code-block:: php

    namespace App\Behat\Context\Transform;

    use Behat\Behat\Context\Context;

    final class ShippingMethodContext implements Context
    {
        /**
         * @var ShippingMethodRepositoryInterface
         */
        private $shippingMethodRepository;

        /**
         * @param ShippingMethodRepositoryInterface $shippingMethodRepository
         */
        public function __construct(ShippingMethodRepositoryInterface $shippingMethodRepository)
        {
            $this->shippingMethodRepository = $shippingMethodRepository;
        }

        /**
         * @Transform :shippingMethod
         */
        public function getShippingMethodByName($shippingMethodName)
        {
            $shippingMethod = $this->shippingMethodRepository->findOneByName($shippingMethodName);
            if (null === $shippingMethod) {
                throw new \Exception('Shipping method with name "'.$shippingMethodName.'" does not exist');
            }

            return $shippingMethod;
        }
    }

.. code-block:: php

    namespace App\Behat\Context\Ui\Admin;

    use App\Behat\Page\Admin\ShippingMethod\UpdatePageInterface;
    use Behat\Behat\Context\Context;

    final class ShippingMethodContext implements Context
    {
        /**
         * @var UpdatePageInterface
         */
        private $updatePage;

        /**
         * @param UpdatePageInterface $updatePage
         */
        public function __construct(UpdatePageInterface $updatePage)
        {
            $this->updatePage = $updatePage;
        }

        /**
         * @Given I want to modify a shipping method :shippingMethod
         */
        public function iWantToModifyAShippingMethod(ShippingMethodInterface $shippingMethod)
        {
            $this->updatePage->open(['id' => $shippingMethod->getId()]);
        }
    }

.. warning::
    Contexts should have single responsibility and this segregation (Setup, Transformer, Ui, etc...) is not accidental.
    We shouldn't create objects in transformer contexts.

Setup contexts
~~~~~~~~~~~~~~

For setup context we need different scenario with more background steps and all preparing scene steps.
Editing scenario will be great for this example:

Scenario::

    Given the store has disabled country "France"
    And I want to edit this country
    When I enable it
    And I save my changes
    Then I should be notified that it has been successfully edited
    And this country should be enabled

.. code-block:: php

    namespace App\Behat\Context\Setup;

    use Behat\Behat\Context\Context;

    final class GeographicalContext implements Context
    {
        /**
         * @var SharedStorageInterface
         */
        private $sharedStorage;

        /**
         * @var FactoryInterface
         */
        private $countryFactory;

        /**
         * @var RepositoryInterface
         */
        private $countryRepository;

        /**
         * @var CountryNameConverterInterface
         */
        private $countryNameConverter;

        /**
         * @param SharedStorageInterface $sharedStorage
         * @param FactoryInterface $countryFactory
         * @param RepositoryInterface $countryRepository
         * @param CountryNameConverterInterface $countryNameConverter
         */
        public function __construct(
            SharedStorageInterface $sharedStorage,
            FactoryInterface $countryFactory,
            RepositoryInterface $countryRepository,
            CountryNameConverterInterface $countryNameConverter
        ) {
            $this->sharedStorage = $sharedStorage;
            $this->countryFactory = $countryFactory;
            $this->countryRepository = $countryRepository;
            $this->countryNameConverter = $countryNameConverter;
        }

        /**
         * @Given /^the store has disabled country "([^"]*)"$/
         */
        public function theStoreHasDisabledCountry($countryName) // This method save country in data base.
        {
            $country = $this->createCountryNamed(trim($countryName));
            $country->disable();

            $this->sharedStorage->set('country', $country);
            // Shared storage is an helper service for transferring objects between steps.
            // There is also SharedStorageContext which use this helper service to transform sentences like "(this country), (it), (its), (theirs)" into Country Object.

            $this->countryRepository->add($country);
        }

        /**
         * @param string $name
         *
         * @return CountryInterface
         */
        private function createCountryNamed($name)
        {
            /** @var CountryInterface $country */
            $country = $this->countryFactory->createNew();
            $country->setCode($this->countryNameConverter->convertToCode($name));

            return $country;
        }
    }
