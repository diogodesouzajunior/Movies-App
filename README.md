# Movies App

## Introdução

Para aplicar os conceitos do Clean Architecture e SOLID, eu estruturaria o projeto utilizando a arquitetura limpa (Clean Architecture) e o gerenciamento de estado com Bloc. Esta abordagem me permitiria criar um código mais modular, escalável e fácil de testar.

## Estrutura do Projeto

Eu estruturaria o projeto da seguinte forma:

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

## Detalhamento da Estrutura

### 1. Core

Para o diretório `core`, eu colocaria código genérico e utilitário que pode ser reutilizado em várias partes do aplicativo.

- **errors**: Eu definiria classes de erro/exceção.
- **usecases**: Eu implementaria casos de uso comuns.
- **utils**: Eu utilizaria utilitários e funções auxiliares.
- **widgets**: Eu criaria widgets reutilizáveis.

### 2. Features

Para cada funcionalidade do aplicativo, eu criaria um módulo separado que segue a arquitetura limpa.

- **data**: Eu implementaria a lógica de dados.
  - **datasources**: Eu implementaria as fontes de dados (APIs, locais, etc.).
  - **models**: Eu criaria modelos de dados que representam a estrutura de dados usada nas fontes de dados.
  - **repositories**: Eu implementaria os repositórios que concretizam os contratos definidos na camada de domínio.

- **domain**: Eu colocaria a lógica de negócio pura.
  - **entities**: Eu criaria as entidades que são objetos de negócio do domínio.
  - **repositories**: Eu definiria os contratos que especificam os métodos que a camada de dados deve implementar.
  - **usecases**: Eu encapsularia a lógica de aplicação em casos de uso.

- **presentation**: Eu colocaria o código de apresentação (UI) e gerenciamento de estado.
  - **blocs**: Eu gerenciaria o estado utilizando Blocs e Cubits.
  - **pages**: Eu criaria as páginas da interface do usuário.
  - **widgets**: Eu criaria widgets específicos da funcionalidade.

## Exemplo de Implementação

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
    // Implementação da chamada de API para obter filmes
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
  // Outros atributos

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

## Injeção de Dependências

Eu usaria o pacote `get_it` para gerenciar as dependências.

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

## Configuração no `main.dart`

Eu configuraria a injeção de dependências no arquivo `main.dart`.

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

Ao seguir essa estrutura, eu conseguiria um projeto organizado e modular, facilitando a manutenção e a expansão do aplicativo conforme necessário.
