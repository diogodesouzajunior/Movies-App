
# Movies App

## Introduction

To apply the concepts of Clean Architecture and SOLID principles, I would structure the project using Clean Architecture and state management with Bloc. This approach would allow me to create a more modular, scalable, and testable codebase.

## Project Structure

I would structure the project as follows:

```
lib/
├── core/
│   ├── errors/
│   ├── usecases/
│   ├── utils/
│   └── widgets/
├── features/
│   ├── movie/
│   │   ├── data/
│   │   │   ├── datasources/
│   │   │   ├── models/
│   │   │   └── repositories/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── repositories/
│   │   │   └── usecases/
│   │   └── presentation/
│   │       ├── blocs/
│   │       ├── pages/
│   │       └── widgets/
├── injection_container.dart
└── main.dart
```

## Structure Details

### 1. Core

In the `core` directory, I would place generic and utility code that can be reused across different parts of the app.

- **errors**: I would define error/exception classes.
- **usecases**: I would implement common use cases.
- **utils**: I would add utilities and helper functions.
- **widgets**: I would create reusable widgets.

### 2. Features

For each feature of the app, I would create a separate module that follows Clean Architecture principles.

- **data**: This is where I would implement data logic.
  - **datasources**: I would implement data sources (APIs, local sources, etc.).
  - **models**: I would create data models representing the data structure used by the data sources.
  - **repositories**: I would implement repositories that concretely realize the contracts defined in the domain layer.

- **domain**: This is where I would put the pure business logic.
  - **entities**: I would create entities that represent business objects in the domain.
  - **repositories**: I would define the contracts specifying the methods that the data layer must implement.
  - **usecases**: I would encapsulate the application logic into use cases.

- **presentation**: This is where I would handle the UI code and state management.
  - **blocs**: I would manage state using Blocs and Cubits.
  - **pages**: I would create user interface pages.
  - **widgets**: I would create feature-specific widgets.

## Implementation Example

### 1. Data Layer

**Datasource**

```dart
// lib/features/movie/data/datasources/movie_remote_datasource.dart

import 'package:movies_app/features/movie/data/models/movie_model.dart';

abstract class MovieRemoteDatasource {
  Future<List<MovieModel>> getMovies();
}

class MovieRemoteDatasourceImpl implements MovieRemoteDatasource {
  @override
  Future<List<MovieModel>> getMovies() {
    // API call implementation to fetch movies
  }
}
```

**Repository Implementation**

```dart
// lib/features/movie/data/repositories/movie_repository_impl.dart

import 'package:movies_app/features/movie/data/datasources/movie_remote_datasource.dart';
import 'package:movies_app/features/movie/domain/entities/movie.dart';
import 'package:movies_app/features/movie/domain/repositories/movie_repository.dart';

class MovieRepositoryImpl implements MovieRepository {
  final MovieRemoteDatasource remoteDatasource;

  MovieRepositoryImpl({required this.remoteDatasource});

  @override
  Future<List<Movie>> getMovies() async {
    final movieModels = await remoteDatasource.getMovies();
    return movieModels.map((model) => model.toEntity()).toList();
  }
}
```

### 2. Domain Layer

**Entity**

```dart
// lib/features/movie/domain/entities/movie.dart

class Movie {
  final int id;
  final String title;
  final String overview;
  // Other attributes

  Movie({
    required this.id,
    required this.title,
    required this.overview,
  });
}
```

**Repository Contract**

```dart
// lib/features/movie/domain/repositories/movie_repository.dart

import 'package:movies_app/features/movie/domain/entities/movie.dart';

abstract class MovieRepository {
  Future<List<Movie>> getMovies();
}
```

**Use Case**

```dart
// lib/features/movie/domain/usecases/get_movies.dart

import 'package:movies_app/core/usecases/usecase.dart';
import 'package:movies_app/features/movie/domain/entities/movie.dart';
import 'package:movies_app/features/movie/domain/repositories/movie_repository.dart';

class GetMovies implements UseCase<List<Movie>, NoParams> {
  final MovieRepository repository;

  GetMovies(this.repository);

  @override
  Future<List<Movie>> call(NoParams params) async {
    return await repository.getMovies();
  }
}
```

### 3. Presentation Layer

**Bloc**

```dart
// lib/features/movie/presentation/blocs/movie_bloc.dart

import 'package:bloc/bloc.dart';
import 'package:equatable/equatable.dart';
import 'package:movies_app/features/movie/domain/entities/movie.dart';
import 'package:movies_app/features/movie/domain/usecases/get_movies.dart';

part 'movie_event.dart';
part 'movie_state.dart';

class MovieBloc extends Bloc<MovieEvent, MovieState> {
  final GetMovies getMovies;

  MovieBloc({required this.getMovies}) : super(MovieInitial()) {
    on<GetMoviesEvent>((event, emit) async {
      emit(MovieLoading());
      final movies = await getMovies(NoParams());
      emit(MovieLoaded(movies: movies));
    });
  }
}
```

**UI**

```dart
// lib/features/movie/presentation/pages/movie_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:movies_app/features/movie/presentation/blocs/movie_bloc.dart';

class MoviePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Movies'),
      ),
      body: BlocBuilder<MovieBloc, MovieState>(
        builder: (context, state) {
          if (state is MovieLoading) {
            return Center(child: CircularProgressIndicator());
          } else if (state is MovieLoaded) {
            return ListView.builder(
              itemCount: state.movies.length,
              itemBuilder: (context, index) {
                return ListTile(
                  title: Text(state.movies[index].title),
                );
              },
            );
          } else {
            return Center(child: Text('Failed to fetch movies'));
          }
        },
      ),
    );
  }
}
```

## Dependency Injection

I would use the `get_it` package to manage dependencies.

```dart
// lib/injection_container.dart

import 'package:get_it/get_it.dart';
import 'package:movies_app/features/movie/data/datasources/movie_remote_datasource.dart';
import 'package:movies_app/features/movie/data/repositories/movie_repository_impl.dart';
import 'package:movies_app/features/movie/domain/repositories/movie_repository.dart';
import 'package:movies_app/features/movie/domain/usecases/get_movies.dart';
import 'package:movies_app/features/movie/presentation/blocs/movie_bloc.dart';

final sl = GetIt.instance;

void init() {
  // Blocs
  sl.registerFactory(() => MovieBloc(getMovies: sl()));

  // Use Cases
  sl.registerLazySingleton(() => GetMovies(sl()));

  // Repositories
  sl.registerLazySingleton<MovieRepository>(
    () => MovieRepositoryImpl(remoteDatasource: sl()),
  );

  // Data Sources
  sl.registerLazySingleton<MovieRemoteDatasource>(
    () => MovieRemoteDatasourceImpl(),
  );
}
```

## Setup in `main.dart`

I would configure dependency injection in the `main.dart` file.

```dart
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:movies_app/injection_container.dart' as di;
import 'package:movies_app/features/movie/presentation/blocs/movie_bloc.dart';
import 'package:movies_app/features/movie/presentation/pages/movie_page.dart';

void main() {
  di.init();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(create: (_) => di.sl<MovieBloc>()),
      ],
      child: MaterialApp(
        title: 'Movies App',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: MoviePage(),
      ),
    );
  }
}
```

By following this structure, I would achieve a well-organized and modular project, making it easier to maintain and expand the app as needed.
