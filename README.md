# django-test-plus
Useful additions to Django's default TestCase from [Revolution Systems](http://www.revsys.com/)

[![travis ci status image](https://secure.travis-ci.org/revsys/django-test-plus.png)](http://travis-ci.org/revsys/django-test-plus) [![Coverage Status](https://coveralls.io/repos/revsys/django-test-plus/badge.svg?branch=master)](https://coveralls.io/r/revsys/django-test-plus?branch=master)

## Rationale

Let's face it, writing tests isn't always fun.  Part of the reason for that is all of the boilerplate you end up writing.  django-test-plus is an attempt to cut down on some of that when writing Django tests. We guarantee it will increase the time before you get carpal tunnel by at least 3 weeks!

## Support

Supports: Python 2 and Python 3

Supports Django Versions: 1.4, 1.5, 1.6, 1.7, and 1.8

## Usage

Using django-test-plus is pretty easy, simply have your tests inherit from test_plus.test.TestCase rather than the normal django.test.TestCase like so:

    from test_plus.test import TestCase

    class MyViewTests(TestCase):
        ...

This is sufficient to get things rolling, but you are encouraged to create *your own* sub-class on a per project basis.  This will allow you to add your own project specific helper methods.

For example, if you have a django project named 'myproject', you might create
the following in ```myproject/test.py```:

    from test_plus.test import TestCase as PlusTestCase

    class TestCase(PlusTestCase):
        pass

And then in your tests use:

    from myproject.test import TestCase

    class MyViewTests(TestCase):
        ...

## Methods

### reverse(url_name, **args, ***kwargs)

When testing views you often find yourself needing to reverse the URL's name. With django-test-plus there is no need for the ```from django.core.urlresolvesr import reverse``` boilerplate.  Instead just use:

    def test_something(self):
        url = self.reverse('my-url-name')
        slug_url = self.reverse('name-takes-a-slug', slug='my-slug')
        pk_url self.reverse('name-takes-a-pk', pk=12)

As you can see our reverse also passes along any args or kwargs you need to pass in.


### get(url_name, **args, ***kwargs)

Another thing you do often is HTTP get urls, our ```get()``` method assumes you are passing in a named URL with any arguments necessary:

    def test_get_named_url(self):
        response = self.get('my-url-name')

### post(url_name, data, **args, ***kwargs)

Our ```post()``` method takes a named URL, the dictionary of data you wish to post and any args or kwargs necessary to reverse the url_name.


    def test_post_named_url(self):
        response = self.post('my-url-name', data={'coolness-factor': 11.0})

### response_XXX(response) - status code checking

Another test you often need to do is check that a response has a certain HTTP status code.  With Django's default TestCase you would write:

    from django.core.urlresolvers import reverse

    def test_status(self):
        response = self.client.get(reverse('my-url-name'))
        self.assertEqual(response.status_code, 200)

With django-test-plus you can shorten that to be:

    def test_better_status(self):
        response = self.get('my-url-name')
        self.response_200(response)

django-test-plus provides the following response method checks for you:

    - response_200()
    - response_201()
    - response_302()
    - response_403()
    - response_404()

All of which take a Django test client response as their only argument.

### get_check_200(url_name, **args, ***kwargs)

GETing and checking views return status 200 is so common a test this method makes it even easier:

    def test_even_better_status(self):
        response = self.get_check_200('my-url-name')

### make_user(username)

When testing out views you often need to create various users to ensure all of your logic is safe and sound.  To make this process easier, this method will create a user for you:

    def test_user_stuff(self)
        user1 = self.make_user('u1')
        user2 = self.make_user('u2')

**NOTE:** This work properly with version of Django prior to 1.6 and will use your own User class if you have created your own User model.

If creating a User in your project is more complicated, say for example you removed the ```username``` field from the default Django Auth model you can provide a [Factory Boy](https://factoryboy.readthedocs.org/en/latest/) factory to create it or simply override this method on your own sub-class.

To use a Factory Boy factory simply create your class like this:

    from test_plus.test import TestCase
    from .factories import UserFactory


    class MySpecialTest(TestCase):
        user_factory = UserFactory

        def test_special_creation(self):
            user1 = self.make_user('u1')

**NOTE:** Users created by this method will always have their password set to the string 'password' in order to ease testing.

## Authentication Helpers

### assertLoginRequired(url_name, **args, ***kwargs)

It's pretty easy to add a new view to a project and forget to restrict it to be login required, this method helps make it easy to test that a given named URL requires auth:

    def test_auth(self):
        self.assertLoginRequired('my-restricted-url')
        self.assertLoginRequired('my-restricted-object', pk=12)
        self.assertLoginRequired('my-restricted-object', slug='something')

### login context

Along with ensuing a view requires login and creating users, the next thing you end up doing is logging in as various users to test our your restriction
logic.  This can be made easier with the following context:

    def test_restrictions(self):
        user1 = self.make_user('u1')
        user2 = self.make_user('u2')

        self.assertLoginRequired('my-protected-view')

        with self.login(username=user1.username, password='password'):
            response = self.get('my-protected-view')
            # Test user1 sees what they should be seeing

        with self.login(username=user2.username, password='password'):
            response = self.get('my-protected-view')
            # Test user2 see what they should be seeing

Since we're likely creating our users using ```make_user()``` from above, we can assume the password is 'password' so the login context assumes that for you as well, so you can do:

    def test_restrictions(self):
        user1 = self.make_user('u1')

        with self.login(username=user1.username):
            response = self.get('my-protected-view')

We can also derive the username if we're using ```make_user()``` so we can shorten that up even further like this:

    def test_restrictions(self):
        user1 = self.make_user('u1')

        with self.login(user1):
            response = self.get('my-protected-view')

## Ensuring low query counts

### assertNumQueriesLessThan(number) - context

Django provides [assertNumQueries](https://docs.djangoproject.com/en/1.8/topics/testing/tools/#django.test.TransactionTestCase.assertNumQueries) which is great when your code generates generates a specific number of queries. However, if due to the nature of your data this number can vary you often don't attempt to ensure the code doesn't start producing a ton more queries than you expect.

**NOTE:** This isn't possible in versions of Django prior to 1.6, so the context will run your code and assertions and issue a warning that it cannot check the number of queries generated.

### assertGoodView(url_name, **args, ***kwargs)

This method does a few of things for you, it:

    - Retrieves the name URL
    - Ensures the view does not generate more than 50 queries
    - Ensures the response has status code 200
    - Returns the response

Often a wide sweeping test like this is better than no test at all. You can use it like this:

    def test_better_than_nothing(self):
        response = self.assertGoodView('my-url-name')
