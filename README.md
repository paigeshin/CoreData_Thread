# CoreData_Thread

# Thread

- ViewContext - UI/Main Thread

### Run on Background

```swift
DispatchQueue.global().async {
    let backgroundContext = CoreDataManager.shared.persistentContainer.newBackgroundContext()
    backgroundContext.perform {
        let movie = Movie(context: backgroundContext)
        movie.title = movieName
        movie.rating = Int16.random(in: 1...5)
        try? backgroundContext.save()
    }
}
```

```swift
private func loadMovies() {
    let request: NSFetchRequest<Movie> = Movie.fetchRequest()
    do {
        self.movies = try CoreDataManager.shared.viewContext.fetch(request)
    } catch {
        print(error.localizedDescription)
    }
}

private func saveMovie(completion: @escaping() -> Void) {
    DispatchQueue.global().async {
        let backgroundContext = CoreDataManager.shared.persistentContainer.newBackgroundContext()
        backgroundContext.perform {
            let movie = Movie(context: backgroundContext)
            movie.title = movieName
            movie.rating = Int16.random(in: 1...5)
            try? backgroundContext.save()
            DispatchQueue.main.async {
                completion()
            }
        }
    }
}
```

### ObjectID is safe within any thread

```swift
private func loadAllMovies(byRating rating: Int, in context: NSManagedObjectContext) -> [Movie] {
    var movies: [Movie]? = []
//        context.perform {
//
//        }
    context.performAndWait {
        let request: NSFetchRequest<Movie> = Movie.fetchRequest()
        request.predicate = NSPredicate(format: "%K > %i", #keyPath(Movie.rating), rating)
        movies = try? context.fetch(request)
    }
    return movies ?? []
}

DispatchQueue.global().async {
      let viewContext = CoreDataManager.shared.viewContext
      let movies = self.loadAllMovies(byRating: 1, in: viewContext)
      let bgContext = CoreDataManager.shared.persistentContainer.newBackgroundContext()
      for movie in movies {
          print(movie.objectID)
          // print(movie.title) // error
          
          bgContext.perform {
              let movie = try? bgContext.existingObject(with: movie.objectID) as? Movie
              if let movie {
                  print(movie.title) // safe 
              }
          }
      }
      
  }
```

### performBackgroundTask

```swift
private func loadAllMovies(completion: @escaping([Movie]) -> Void) {
    CoreDataManager.shared.persistentContainer.performBackgroundTask { context in
        let request: NSFetchRequest<Movie> = Movie.fetchRequest()
        guard let movies = try? context.fetch(request) else {
            return
        }
        DispatchQueue.main.async {
            let viewContext = CoreDataManager.shared.viewContext
            let movies = movies.compactMap { movie in
                try? viewContext.existingObject(with: movie.objectID) as? Movie
            }
            completion(movies)
        }
    }
}

self.loadAllMovies { movies in
    self.movies = movies
}
```

### Subscribing to Context Change Notifications in Core Data

```swift
class MovieListViewModel: ObservableObject {
    
    @Published var movies: [MovieViewModel] = []
    
    init() {
        let didSaveNotification = NSManagedObjectContext.didSaveObjectsNotification
//        let context = CoreDataManager.shared.persistentContainer.newBackgroundContext()
        let context = CoreDataManager.shared.persistentContainer.viewContext
        NotificationCenter.default.addObserver(self, selector: #selector(didSave(_:)), name: didSaveNotification, object: context)
    }
    
    @objc func didSave(_ notification: Notification) {
        let insertedObjectsKey = NSManagedObjectContext.NotificationKey.insertedObjects.rawValue
        print(notification.userInfo?[insertedObjectsKey])
        self.loadMovies()
    }
    
    func saveMovieOnBackground(title: String, rating: Int16) {
        CoreDataManager.shared.persistentContainer.performBackgroundTask { context in
            let movie = Movie(context: context)
            movie.title = title
            movie.rating = rating
            try? context.save()
        }
    }
    
    func saveMoveOnMain(title: String, rating: Int16) {
        let viewContext = CoreDataManager.shared.viewContext
        let movie = Movie(context: viewContext)
        movie.title = title
        movie.rating = rating
        try? viewContext.save()
    }
    
    func loadMovies() {
        let request: NSFetchRequest<Movie> = Movie.fetchRequest()
        
        do {
            self.movies = try CoreDataManager.shared.viewContext.fetch(request).map(MovieViewModel.init)
        } catch {
            print(error.localizedDescription)
        }
    }
    
}

struct MovieViewModel {
    
    let movie: Movie
    let id = UUID()
    
    var title: String {
        return self.movie.title ?? ""
    }
    
    var rating: Int16 {
        return self.movie.rating
    }
    
}
```

### Merge Changes

```swift
@objc func didSave(_ notification: Notification) {
    let viewContext = CoreDataManager.shared.viewContext
    DispatchQueue.main.async {
        // background -> main
        viewContext.mergeChanges(fromContextDidSave: notification)
    }
}
```

### Merge Changes on configuration

```swift
// do some tasks on background ⇒ on main
persistentContainer.viewContext.automaticallyMergesChangesFromParent = true 
```

```swift
import Foundation
import CoreData

class CoreDataManager {
    
    static let shared = CoreDataManager()
    let persistentContainer: NSPersistentContainer
    private var _backgroudContext: NSManagedObjectContext?
    
    var viewContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
    
    var backgroundContext: NSManagedObjectContext {
        if _backgroudContext == nil {
            _backgroudContext = persistentContainer.newBackgroundContext()
        }
        return _backgroudContext!
    }
        
    private init() {
        persistentContainer = NSPersistentContainer(name: "MovieAppModel")
        persistentContainer.viewContext.automaticallyMergesChangesFromParent = true 
        persistentContainer.loadPersistentStores { (description, error) in
            if let error = error {
                fatalError("Unable to initialize Core Data \(error)")
            }
        }
    }
    
}
```
