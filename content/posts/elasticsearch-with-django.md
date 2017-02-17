Title: Elasticsearch with Django
Date: 2016-01-07 17:00
Category: django
tags: django, elasticsearch, python

## Elasticsearch with Django

I've been meaning to add search functionality to complete the first part of a project I've been working on. The project in
question is a database of homebrew materials. At the same time, I've also just switched from blogger to a Pelican based python implementation and needed to write a new article. This may be considered a naive implementation, but after some research I decided that elasticsearch worked best for me for the following reasons:

* Scalability: While my records of hops, yeasts and grains wouldn't be much to require such a high powered search, I knew that I would need a solution that would scale and elasticsearch can handle small records when I need them, but could handle large records when the time comes.

* Ease: I've found the elasticsearch online documentation to be quite easy to read. The "Definitive Guide" online was easy enough to follow and complete a few simple curl commands for easy record addition, searching, updating and deleting.

* Compatibility: I had spent quite a bit of time on IRC asking questions about search functionality and reading other implementations online. Haystack seemed to be the obvious answer at first, but given that I had heard that this library can be difficult depending on the backend. After a few other library evaluations, I settled with elasticsearch-py, a lower level implementation that offered easy and simple access to the engine.

The elasticsearch-py library allows me to initialize one object that automatically uses the default elasticsearch engine connection,
localhost:9200--

    from elasticsearch import Elasticsearch
    es = Elasticsearch()

I'll also be using the `IndicesClient` for managing indices and takes the Elasticsearch client as an argument--

    from elasticsearch.client import IndicesClient
    es_client = IndicesClient(client=es)

### Implementation

Search functions seemed relatively simple when I first decided to add the feature to my site. But there turned out to be so many different ways to implment search that planning an implementation seemed a little overwhelming. One source I relied on for some bare bones guidance was (Myklyta Kliestov's excellent article on Elasticsearch and Django)[https://qbox.io/blog/how-to-elasticsearch-python-django-part1]. I wasn't able to follow step by step as he recommends some code changes that I didn't agree with (for example, he recommends overriding a Django setting to allow custom meta fields and I just wasn't ready to take that leap), but it definitely gave me some great ideas. I also prefer to follow a test driven approach so I took some breaks to think about how I was going to test my code prior to writing it.

### A Basic Django Base Command

Writing a base command seemed like a good place to start. Base commands allow you to run `manage.py` commands from the command line similar to `runserver`, `makemigrations`, etc. My base command follows Kliestov's approach with a few variations that I won't get into. But before I share that code, I'd like to stick with TDD principles and share my test first--

    from django.core.management import call_command
    from django.test import TestCase
    from django.utils.six import StringIO

    from elasticsearch import Elasticsearch
    from elasticsearch.client import IndicesClient


    class TestSearch(TestCase):

        def setUp(self):
            self.es = Elasticsearch()
            self.client = IndicesClient(client=self.es)

        def test_push_to_index_creates_index(self):
            self.out = StringIO()
            call_command('push_hop_to_index', stdout=self.out)
            self.es_hop_index = self.client.get_mapping(index='hop')['hop']['mappings']['hop']['properties']

            self.id_type = self.es_hop_index['id']['type']
            self.name_type = self.es_hop_index['name']['type']
            self.min_alpha_acid_type = self.es_hop_index['min_alpha_acid']['type']
            self.max_alpha_acid_type = self.es_hop_index['max_alpha_acid']['type']
            self.country_type = self.es_hop_index['country']['type']
            self.comments_type = self.es_hop_index['comments']['type']

            self.assertEqual(self.id_type, 'long')
            self.assertEqual(self.name_type, 'string')
            self.assertEqual(self.min_alpha_acid_type, 'double')
            self.assertEqual(self.max_alpha_acid_type, 'double')
            self.assertEqual(self.country_type, 'string')
            self.assertEqual(self.comments_type, 'string')


The test is simple: call the base command using Django's built in `call_command`, then use `elasticsearch-py` to query elastic search to ensure that the index mappings appear as expected. I added a few records to the Elasticsearch index when I was first testing out the API. Elasticsearch generally tries to guess a document's mapping based on the field values, so I had a mapping to use but figured that being explicit was best to remove any guess work. Let's take a look at the base command itself


* app/management/commands/push_to_index.py

    from elasticsearch.client import IndicesClient
    from elasticsearch import Elasticsearch, RequestsHttpConnection

    from django.core.management.base import BaseCommand

    ES_CLIENT = Elasticsearch(
        ['http://127.0.0.1:9200/'],
        connection_class=RequestsHttpConnection
    )

    es_mapping = {
        'properties': {
            'country': {
                'type': 'string'
                        },
            'id': {
                'type': 'long'
                        },
            'name': {
                'type': 'string'
                        },
            'comments': {
                'type': 'string'
                        },
            'max_alpha_acid': {
                'type': 'double'
                        },
            'min_alpha_acid': {
                'type': 'double'
                    }
                }
            }


    class Command(BaseCommand):

        help = "Command to create hop index in elasticsearch"

        def handle(self, *args, **kwargs):
            self.recreate_index()

        def recreate_index(self):
            indices_client = IndicesClient(client=ES_CLIENT)

            index_name = 'hop'
            if indices_client.exists(index_name):
                indices_client.delete(index=index_name)
            indices_client.create(index=index_name)
            indices_client.put_mapping(
                doc_type='hop',
                body=es_mapping,
                index=index_name
            )

I explicitly setup my Elasticsearch mapping to be consistent, my base command simply checks if the index exists. If it does exist, it gets deleted and recreated and the mapping is updated.

### Saving to Elasticsearch

So I've created a base command to setup the index. Now I'll need to make sure that my app is updating Elasticsearch as it makes changes to the database. If a record gets added, we'll need it to be added to ES; if it's updated, we'll update it and the same for deleting records. I already have an existing test that checks that items are saved and can be retrieved. I've added to it to perform additional checks to Elasticsearch--

    from django.test import TestCase
    from elasticsearch import Elasticsearch

    from homebrewdatabase.models import Hop, Grain, Yeast


    class HopModelTest(TestCase):

        def setUp(self):
            # We have a setup method to initialize the ES client
            self.es_client = Elasticsearch()


        def test_saving_items_and_retrieving_later(self):
            """
            Asserts that a user can successfully enter and retrieve a hop record using the Hop model
                    :return: pass or fail
            """

            first_hop = Hop()
            first_hop.name = 'Amarillo'
            first_hop.min_alpha_acid = '8.00'
            first_hop.max_alpha_acid = '11.00'
            first_hop.country = 'USA'
            first_hop.comments = 'Pretty good, all around'

            # Capture the object returned by the save method
            first_result = first_hop.save()

            second_hop = Hop()
            second_hop.name = 'Chinook'
            second_hop.min_alpha_acid = '12.00'
            second_hop.max_alpha_acid = '14.00'
            second_hop.country = 'USA'
            second_hop.comments = 'Good for bittering, not great for aroma'
            second_result = second_hop.save()

            saved_hops = Hop.objects.all()
            self.assertEqual(saved_hops.count(), 2)

            first_saved_hop = saved_hops[0]
            second_saved_hop = saved_hops[1]
            self.assertEqual(first_saved_hop.name, 'Amarillo')
            self.assertEqual(first_saved_hop.min_alpha_acid, 8.00)
            self.assertEqual(first_saved_hop.max_alpha_acid, 11.00)
            self.assertEqual(first_saved_hop.country, 'USA')
            self.assertEqual(first_saved_hop.comments, 'Pretty good, all around')

            self.assertEqual(second_saved_hop.name, 'Chinook')
            self.assertEqual(second_saved_hop.min_alpha_acid, 12.00)
            self.assertEqual(second_saved_hop.max_alpha_acid, 14.00)
            self.assertEqual(second_saved_hop.country, 'USA')
            self.assertEqual(second_saved_hop.comments, 'Good for bittering, not great for aroma')

            # Query Elasticsearch for a record with a matching id
            first_es_hop_record = self.es_client.get(index="hop", doc_type="hop", id=first_result.id)['_source']
            second_es_hop_record = self.es_client.get(index="hop", doc_type="hop", id=second_result.id)['_source']

            # Verify that the record fields match the input
            self.assertEqual(first_es_hop_record['name'], 'Amarillo')
            self.assertEqual(first_es_hop_record['min_alpha_acid'], 8.00)
            self.assertEqual(first_es_hop_record['max_alpha_acid'], 11.00)
            self.assertEqual(first_es_hop_record['country'], 'USA')
            self.assertEqual(first_es_hop_record['comments'], 'Pretty good, all around')

            self.assertEqual(second_es_hop_record['name'], 'Chinook')
            self.assertEqual(second_es_hop_record['min_alpha_acid'], 12.00)
            self.assertEqual(second_es_hop_record['max_alpha_acid'], 14.00)
            self.assertEqual(second_es_hop_record['country'], 'USA')
            self.assertEqual(second_es_hop_record['comments'], 'Good for bittering, not great for aroma')

I've added comments to the code explaining where the changes have been made, but here's a recap: we initialize the ES client, capture the record returned by the object's save method and use the associated id to query Elastic search for a copy of the record before checking the fields to ensure that the output matches. This test currently fails because there's been no changes made to update Elasticsearch--

    Error
    Traceback (most recent call last):
      File "/Users/antonioalaniz1/workspace/PycharmProjects/hashtagbrews/homebrewdatabase/tests/test_models.py", line 52, in test_saving_items_and_retrieving_later
        first_es_hop_record = self.es_client.get(index="hop", doc_type="hop", id=first_result.id)['_source']
    AttributeError: 'NoneType' object has no attribute 'id'

But we're going to change that. At first I thought that the best way to go about this would be to add some additional lines in my views that update Elasticsearch after calling a `save` method. But after some research, I think I'm going to override my model's `save` and `delete` methods to avoid complicating my views any further. Since my site has will not be pushed live before this feature is implemented, I can be fairly certain that all will be in sync in the beginning (though some scheduled tasks with, say, Celery will probably be needed to check that this is the case).

*models.py*

    def save(self, *args, **kwargs):
        es_client = Elasticsearch()
        super(Hop, self).save(*args, **kwargs)
        if self.pk is not None:
            es_client.index(
                index="hop",
                doc_type="hop",
                id=self.pk,
                body={
                      'name': self.name,
                      'min_alpha_acid': self.min_alpha_acid,
                      'max_alpha_acid': self.max_alpha_acid,
                      'country': self.country,
                      'comments': self.comments
                      },
                refresh=True
            )
        else:
            es_client.create(
                index="hop",
                doc_type="hop",
                id=self.pk,
                body={'name': self.name,
                      'min_alpha_acid': self.min_alpha_acid,
                      'max_alpha_acid': self.max_alpha_acid,
                      'country': self.country,
                      'comments': self.comments
                      },
                refresh=True
            )

This gets my test passing. Time to adjust the view. I already have a suite of tests in place that are checking views. Since the tests just check for matches in the decoded response content, I won't need to adjust them and won't post them here. But I *do* want my views to perform searches from Elasticsearch instead of the database query. The view listing all records of hops probably isn't necessary, but I thought it would be a good first step toward integrating the project. Here's my view--

    def hops(request):
        """
        Main hops page list
                 :param request: Django HttpRequest object
                 :return: renders 'homebrewdatabase/hops.html';

                 * context
                     - 'hops'
                     - 'form'
        """

        # Used to use `hops_list = Hop.objects.all()`

        hops_list = []
        for hit in es_client.search(index='hop')['hits']['hits']:
            entry = hit['_source']
            entry['id'] = hit['_id']
            hops_list.append(entry)
        return render(request, 'homebrewdatabase/hops.html', {'hops': hops_list, 'form': HopForm()})


As you can see, populating my `hops_list` entries just got a little more complicated. I first make the call to get all hits in an empty index search. Since the `_source` key in the resulting dictionary doesn't contain the primary key of the record, I need to go back and add it to make my view work properly. My tests now pass again when adding a new record and searching that it shows up in the all list. I'll need to make one more change to make sure that deleted records are moved from the index before finally implementing the search functionality.

#### A quick note on testing with Elasticsearch

I found the Elasticsearch API easy to work with after I reviewed the docs and played around the the API. But I did run into some issues with testing that required me to constantly delete and recreate my index. Eventually, I worked this into my `setUp` and `tearDown` methods (I also had to delete my existing records that I used on click-through tests)--

    class TestHopsPageView(TestCase):
        """
        Class for testing hops related views
        """

        def setUp(self):
            self.es_client = Elasticsearch()
            call_command('push_hop_to_index')

        def tearDown(self):
            call_command('push_hop_to_index')


### Deleting from Elasticsearch

Removing records from ES was incredibly easy once I got down adding and updating--much the same when setting up these views with Django for the first time. As is the style of this article, I added some entries to my unit test first before overriding my `delete` method on the Hop model--

    def test_delete_hop_record(self):
        """
        Checks that the '/beerdb/delete/%d/hops' url can delete a hop record
                :return: pass or fail
        """

        self.client.post(
            '/beerdb/add/hops/',
            data={
                'name': 'Northern',
                'min_alpha_acid': 18.00,
                'max_alpha_acid': 12.00,
                'country': 'USA',
                'comments': 'Very bitter, not good for aroma'
            })

        hop_instance = Hop.objects.filter(name='Northern')[0]

        # Search for the record in ES
        es_hop_record = self.es_client.get(index='hop', id=hop_instance.id)['_source']

        self.assertEqual(hop_instance.name, 'Northern')

        # Confirm the ES record matches what we're looking for
        self.assertEqual(es_hop_record['name'], 'Northern')

        response = self.client.get('/beerdb/delete/%d/hops/' % hop_instance.id)

        self.assertEqual(response.status_code, 200)

        response = self.client.post('/beerdb/delete/%d/hops/' % hop_instance.id)

        self.assertEqual(response.status_code, 302)

        hop_list = Hop.objects.filter(name='Northern')

        # Look up the record again and verify there are no hits associated with the record
        es_hop_record2 = self.es_client.search(index='hop', body={"query": {"match": {'name': 'Northern'}}})['hits']

        self.assertEqual(len(hop_list), 0)
        self.assertEqual(es_hop_record2['total'], 0)

This test is part of a larger suite. You can see where I call make calls to the `es_client`; in this case, making sure the record exists in ES after adding the record, then verifying that there are no 'hits' when searching for the record. ES provides a `hits` key when using the `search` method to give us a total of how many hits match the search. Tests are now failing as the ES record is still in the index after the record has been deleted in the Django database. To keep consistent with our addition and update methods, we'll overwrite the model's associated `delete` method--

    def delete(self, *args, **kwargs):
        es_client = Elasticsearch()
        hop_id = self.pk
        super(Hop, self).delete(*args, **kwargs)
        es_client.delete(
                index="hop",
                doc_type="hop",
                id=hop_id,
                refresh=True
        )

Tests are passing once again. Time to add the search functionality?--Elasticsearch integrated into views? Check. Tested that we are keeping add, updating and delete functions in sync between the database and Elasticsearch index? Check. All tests are passing? Check. Sounds like I'm good to move forward!

### Adding a Search Function

Before I edit my view, I'm going to write a quick test--

    def test_search_GET_request_returns_matching_results(self):

        self.client.post(      #1
            '/beerdb/add/hops/',
            data={
                'name': 'Cascade',
                'min_alpha_acid': 19.00,
                'max_alpha_acid': 21.00,
                'country': 'USA',
                'comments': 'Very bitter, not good for aroma'
            })

        self.client.post(     #2
            '/beerdb/add/hops/',
            data={
                'name': 'Chinook',
                'min_alpha_acid': 25.00,
                'max_alpha_acid': 31.00,
                'country': 'USA',
                'comments': 'High bitterness, similar to Cascade'
            })

        self.client.post(    #3
            '/beerdb/add/hops/',
            data={
                'name': 'Amarillo',
                'min_alpha_acid': 24.00,
                'max_alpha_acid': 32.00,
                'country': 'USA',
                'comments': 'Very bitter, not good for aroma'
            })

        #4
        request = HttpRequest()
        request.method = 'GET'
        request.GET['query'] = 'Cascade'

        response = hops(request)

        #5
        self.assertIn('Cascade', response.content.decode())
        self.assertIn('High bitterness,', response.content.decode())
        self.assertNotIn('Amarillo', response.content.decode())

The test should be fairly simple to follow, but here's the breakdown--

1, 2, 3 -- We create some test records to store in the database. Since Django creates a blank, development database we need to populate it with some data to search. While writing this part up, it occurred to me that I might start using a factory library to populate the data and take some of the repetition out of my test setups (remember that the `setUp` and `tearDown` methods are still calling the base command that clears out our Elasticsearch index with each test)

4 -- Create a request, add the method attribute ('GET' since this is a search and no data is being posted) and a key to the method with the value we want to search. Note the 'query' key in the `request.GET` dict.

5 -- After sending the response to our view, we search for the hop name 'Cascade' as it's included in the query. I'm also searching for the comment that contains the search term as my Elasticsearch view will search the name, comments and country fields.

This test fails (of course), because we haven't setup the view. While I was working on the test, I also scratched out a rough rewrite of the view. After some playing around, I was able to get the test to pass--

    def hops(request):
        """
        Main hops page list
                 :param request: Django HttpRequest object
                 :return: renders 'homebrewdatabase/hops.html';

                 * context
                     - 'hops'
                     - 'form'
        """

        hops_list = []
        es = Elasticsearch()
        if request.GET.get('query'):
            entries = es.search(index='hop', body={"query": {
                                                        "query_string": {
                                                            "fields": ["name", "country", "comments"],
                                                            "query": request.GET['query']
                                                                        }
                                                            }})['hits']['hits']
        else:
            entries = es_client.search(index='hop')['hits']['hits']

        # Add the id to the source field and build the list of entries
        for entry in entries:
            entry['_source']['id'] = entry['_id']
            entry = entry['_source']
            hops_list.append(entry)
        return render(request, 'homebrewdatabase/hops.html', {'hops': hops_list, 'form': HopForm()})


The view searches for the 'query' key in the request.GET dictionary. If the element is present, the view queries the ES index looking specifically at the name, country and comments fields. If the element is not present, the view queries all entries in the index. Once queried, the record's id (the model primary key) is added to each entry and all are appended to a dictionary to be rendered in my hops view. I run my entire test suite to ensure that nothing is broken after my changes--

    Testing started at 8:43 AM ...
    Creating test database for alias 'default'...
    Destroying test database for alias 'default'...

    Process finished with exit code 0


Success!

### Future Improvements

If I had more time, I would definitely write helper functions that query ES in both my test and views. Since I'll be writing these calls alot and they often require accessing some deeper levels of dictionaries, writing a helper function with a simple description would be much easier for anyone that reviews the code.

