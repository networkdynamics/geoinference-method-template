# Implementing a GIMethod

A model is produced through a training procedure, and then used to infer locations for posts with unknown locations. Dataset access (including to gold standard data) is provided through the methods in dataset.py. 

Each geoinference method must consist of two key classes:
* A subclass of GIMethod. Produces a model from training data. 
* A subclass of GIModel. Infers locations for posts with unknown locations

###GIMethod
GIMethod is the abstract base class for all geographical inference systems supported by the geoinference framework. Any geoinference subclass must explicitly subclass it in the method.py file, and implement its two methods: 
* "GIMethod.train_model(...)" 
  * builds a geoinference model using the settings, dataset, and model_dir provided. The settings can be used by the implementer for greater control over model creation. Dataset selection will be discussed below. The model directory refers to where the model will be stored (pickled), and is most likely the current working directory.
  * This method returns a subclass of GIModel specific to the subclass of GIMethod that has implemented this method. 
* "GIMethod.load_model(...)". 
  * GIMethod.load_model unpickles the created model.

### GIModel
GIModel contains two methods: 

* "GIModel.infer_post_location"
   * Must be implemented
* "GIModel.infer_posts_by_user"
  * May optionally be implemented  
  * By default, GIModel.infer_posts_by_user calls GIModel.infer_post_location for each post in the list (each post being guaranteed to belong to the same user). 

Aside from the restrictions mentioned above, the implementer can structure the package in the way they find most suitable for their method. 
 
## Examples:

###Jakarta
Jakarta is a baseline method that geolocates everyone in Jakarta, Indonesia (the city with the most Twitter users). It represents the absolute minimum of what must be included in a geolocation method.  

    
    from geolocate import GIMethod, GIModel

    JAKARTA_LAT_LON = (-6.2000, 106.8)

    class Jakartr_Model(GIModel):
        '''
            A baseline GIModel that says everyone is in Jakarta, Indonesia, the city
            with the most Twitter users.
            '''

        def __init__(self):
        pass


        def infer_post_location(self, post):
            return JAKARTA_LAT_LON


        def infer_posts_by_user(self, posts):

            # Fill the array with Jakarta's location
            locations = []
            for i in range(0, len(posts)):
                locations.append(JAKARTA_LAT_LON)
            return locations


    class Jakartr(GIMethod):
        def __init__(self):
            pass
    
        def train_model(self, setting, dataset, model_dir):
            # Jakarta is fully unsupervised
            return Jakartr_Model()
    
        def load_model(self, model_dir, settings):
            # Jakarta is fully unsupervised
                return Jakartr_Model()

###Accessing the Dataset - useful methods

The dataset is a collection of posts such that each post belongs to a single user and posts may
mention other users (called "mentions").

dataset.user_iter will return an iterator over all posts in the dataset grouped by user. Each user is represented by a list of their posts. 

    for user in dataset.user_iter():
            user_id = user["user_id"]
            # This strategy returns null if no location was found
            home_loc = self.get_location(user["posts"], posts_to_use, can_use_home_loc)
	
The user_home_location_iter (from the sparse dataset method) returns an iterator over all the users whose home location has already been itentified. The bi_mention_network() returns the undirected mention network for the dataset, consisting only of edges between users who have both mentioned each other.

The code below tries to associate a home location for each of the users in the network. 

        mention_network = dataset.bi_mention_network()
        all_users = set(mention_network.nodes_iter())
        for user_id, home_loc in dataset.user_home_location_iter():           
            if not user_id in all_users:
                continue
            self.user_to_home_loc[user_id] = home_loc

Dataset.py and sparse_dataset.py both contain many similar access methods. 

##Command Line Usage

All functionality is accessed through the geoinf command-line tool (accessed as geoinf.local if the package is not installed for development purposes). For a complete list of functions and options, run

    geoinf -h

The subcommands are structured as:

    geoinf <subcommand> <args>

The core subcommands include:

    build_dataset - builds a new dataset from raw post data
    train - build a new model for a given method
    infer_by_post - instruct the model chosen to infer the location for posts, ordered by post.
    infer_by_user - instruct the model chosen to infer the location for posts, ordered by user.
