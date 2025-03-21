Vou fazer algumas mudanças talvez, testei e não deu muito certo.
### Aqui está um guia passo a passo para desenvolver uma aplicação CRUD de Pokémon usando NestJS com MongoDB (Mongoose) e Swagger, seguindo os princípios SOLID.

# Guia de Aplicação NestJS com MongoDB + Swagger: CRUD de Pokémon

## 1. Configuração inicial do projeto
Primeiro, vamos instalar o NestJS CLI e criar um novo projeto:
````
npm i -g @nestjs/cli

nest new pokemon-api
//Ou se já estiver dentro da pasta 
nest new .
 
cd pokemon-api

code .
````

Instale as dependências necessárias:
````
npm install @nestjs/mongoose mongoose @nestjs/swagger class-validator class-transformer
````

## 2. Configuração do Swagger
Edite o arquivo main.ts:
````
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Pokemon API')
    .setDescription('API para gerenciamento de Pokémons')
    .setVersion('1.0')
    .addTag('pokemon')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
````

## 3. Criação do módulo Pokemon
Crie a estrutura de diretórios:
````
nest g module pokemon
nest g controller pokemon
nest g service pokemon
````

## 4. Configuração do MongoDB
Edite o arquivo app.module.ts:
````
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PokemonModule } from './pokemon/pokemon.module';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/pokemon-db'),
    PokemonModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
````

## 5. Enum de Types
Crie o arquivo pokemon/enum/pokemon-type.enum.ts:
````
export enum PokemonType {
  NORMAL = 'NORMAL',
  FIRE = 'FIRE',
  WATER = 'WATER',
  ELECTRIC = 'ELECTRIC',
  GRASS = 'GRASS',
  ICE = 'ICE',
  FIGHTING = 'FIGHTING',
  POISON = 'POISON',
  GROUND = 'GROUND',
  FLYING = 'FLYING',
  PSYCHIC = 'PSYCHIC',
  BUG = 'BUG',
  ROCK = 'ROCK',
  GHOST = 'GHOST',
  DRAGON = 'DRAGON',
  DARK = 'DARK',
  STEEL = 'STEEL',
  FAIRY = 'FAIRY',
}
````

## 6. Schema do Pokémon (MongoDB)
Crie o arquivo pokemon/schemas/pokemon.schema.ts:
````
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';
import { PokemonType } from '../enum/pokemon-type.enum';

export type PokemonDocument = Pokemon & Document;

@Schema()
export class Pokemon {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true })
  number: number;

  @Prop({ type: [String], enum: PokemonType, required: true })
  types: PokemonType[];

  @Prop()
  description: string;
}

export const PokemonSchema = SchemaFactory.createForClass(Pokemon);
````

## 7. DTOs (Data Transfer Objects)
Crie os arquivos de DTOs:
pokemon/dto/create-pokemon.dto.ts:
````
import { ApiProperty } from '@nestjs/swagger';
import { PokemonType } from '../enum/pokemon-type.enum';

export class CreatePokemonDto {
  @ApiProperty({ description: 'Nome do Pokémon' })
  name: string;

  @ApiProperty({ description: 'Número do Pokémon na Pokédex' })
  number: number;

  @ApiProperty({ 
    description: 'Tipos do Pokémon', 
    enum: PokemonType, 
    isArray: true 
  })
  types: PokemonType[];

  @ApiProperty({ description: 'Descrição do Pokémon' })
  description: string;
}
````

E pokemon/dto/update-pokemon.dto.ts:
````
import { ApiProperty } from '@nestjs/swagger';
import { PokemonType } from '../enum/pokemon-type.enum';

export class UpdatePokemonDto {
  @ApiProperty({ description: 'Nome do Pokémon', required: false })
  name?: string;

  @ApiProperty({ description: 'Número do Pokémon na Pokédex', required: false })
  number?: number;

  @ApiProperty({ 
    description: 'Tipos do Pokémon', 
    enum: PokemonType, 
    isArray: true,
    required: false 
  })
  types?: PokemonType[];

  @ApiProperty({ description: 'Descrição do Pokémon', required: false })
  description?: string;
}
````

## 8. Repository Pattern
Crie a interface do repositório pokemon/repository/pokemon-repository.interface.ts:
````
import { CreatePokemonDto } from "../dto/create-pokemon.dto";
import { UpdatePokemonDto } from "../dto/update-pokemon.dto";
import { Pokemon } from "../schemas/pokemon.schema";

export abstract class PokemonRepository {
    abstract findAll(): Promise<Pokemon[] | null>;
    abstract findById(id: string): Promise<Pokemon | null>;
    abstract findByNumber(number: number): Promise<Pokemon | null>;
    abstract create(createPokemonDto: CreatePokemonDto): Promise<Pokemon>;
    abstract update(id: string, updatePokemonDto: UpdatePokemonDto): Promise<Pokemon | null>;
    abstract remove(id: string): Promise<Pokemon | null>;
}
````

Agora, crie a implementação do repositório pokemon/repository/pokemon-repository.mongodb.ts:
````
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { Pokemon, PokemonDocument } from '../schemas/pokemon.schema';
import { CreatePokemonDto } from '../dto/create-pokemon.dto';
import { UpdatePokemonDto } from '../dto/update-pokemon.dto';
import { PokemonRepository } from './pokemon-repository.interface';

@Injectable()
export class PokemonMongoRepository implements PokemonRepository {
  constructor(
    @InjectModel(Pokemon.name) private pokemonModel: Model<PokemonDocument>,
  ) {}

  async findAll(): Promise<Pokemon[]> {
    return this.pokemonModel.find().exec();
  }

  async findById(id: string): Promise<Pokemon | null> {
    return this.pokemonModel.findById(id).exec();
  }

  async findByNumber(number: number): Promise<Pokemon | null> {
    return this.pokemonModel.findOne({ number }).exec();
  }

  async create(createPokemonDto: CreatePokemonDto): Promise<Pokemon> {
    const newPokemon = new this.pokemonModel(createPokemonDto);
    return newPokemon.save();
  }

  async update(id: string, updatePokemonDto: UpdatePokemonDto): Promise<Pokemon | null> {
    return this.pokemonModel.findByIdAndUpdate(id, updatePokemonDto, { new: true }).exec();
  }

  async remove(id: string): Promise<Pokemon | null> {
    return this.pokemonModel.findByIdAndDelete(id).exec();
  }
}
````

Agora, crie a implementação do repositório pokemon/pokemon.constants.ts:
````
export const POKEMON_REPOSITORY = 'POKEMON_REPOSITORY';
```` 

## 9. Service
Edite o arquivo pokemon/pokemon.service.ts:
````
import { Inject, Injectable, NotFoundException } from '@nestjs/common';
import { PokemonRepository } from './repository/pokemon-repository.interface';
import { CreatePokemonDto } from './dto/create-pokemon.dto';
import { UpdatePokemonDto } from './dto/update-pokemon.dto';

@Injectable()
export class PokemonService {

    constructor(
      @Inject('POKEMON_REPOSITORY')
      private readonly pokemonRepository: PokemonRepository) {}

    async create(createPokemonDto: CreatePokemonDto) {
        return this.pokemonRepository.create(createPokemonDto);
    }

    async findAll() {
        return this.pokemonRepository.findAll();
    }

    async findOne(id: string) {
        const pokemon = await this.pokemonRepository.findById(id);
        if(!pokemon){
            throw new NotFoundException(`Pokemon #${id} not found`);
        }
        return pokemon;
    }

    async findByNumber(number: number) {
        const pokemon = await this.pokemonRepository.findByNumber(number);
        if (!pokemon) {
          throw new NotFoundException(`Pokemon #${number} not found`);
        }
        return pokemon;
      }
    
    async update(id: string, updatePokemonDto: UpdatePokemonDto) {
        const pokemon = await this.pokemonRepository.update(id, updatePokemonDto);
        if (!pokemon) {
          throw new NotFoundException(`Pokemon #${id} not found`);
        }
        return pokemon;
      }
    
      async remove(id: string) {
        const pokemon = await this.pokemonRepository.remove(id);
        if (!pokemon) {
          throw new NotFoundException(`Pokemon #${id} not found`);
        }
        return pokemon;
      }
}
````

## 10. Controller
Edite o arquivo pokemon/pokemon.controller.ts:
````
import { Body, Controller, Delete, Get, Param, Patch, Post } from '@nestjs/common';
import { PokemonService } from './pokemon.service';
import { CreatePokemonDto } from './dto/create-pokemon.dto';
import { ApiOperation, ApiParam, ApiResponse } from '@nestjs/swagger';
import { UpdatePokemonDto } from './dto/update-pokemon.dto';

@Controller('pokemon')
export class PokemonController {

    constructor(private readonly pokemonService: PokemonService) {}

    @Post()
    @ApiOperation({ summary: 'Criar um novo Pokémon' })
    @ApiResponse({ status: 201, description: "Pokemon criado com sucesso." })
    async create(@Body() CreatePokemonDto: CreatePokemonDto) {
        return await this.pokemonService.create(CreatePokemonDto);
    }

    @Get()
    @ApiOperation({ summary: 'Listar todos os Pokémons' })
    @ApiResponse({ status: 200, description: 'Lista de Pokémons retornada com sucesso.' })
    async findAll() {
      return await this.pokemonService.findAll();
    }
  
    @Get(':id')
    @ApiOperation({ summary: 'Buscar um Pokémon pelo ID' })
    @ApiParam({ name: 'id', description: 'ID do Pokémon' })
    @ApiResponse({ status: 200, description: 'Pokémon encontrado com sucesso.' })
    @ApiResponse({ status: 404, description: 'Pokémon não encontrado.' })
    async findOne(@Param('id') id: string) {
      return await this.pokemonService.findOne(id);
    }
  
    @Get('number/:number')
    @ApiOperation({ summary: 'Buscar um Pokémon pelo número na Pokédex' })
    @ApiParam({ name: 'number', description: 'Número do Pokémon na Pokédex' })
    @ApiResponse({ status: 200, description: 'Pokémon encontrado com sucesso.' })
    @ApiResponse({ status: 404, description: 'Pokémon não encontrado.' })
    async findByNumber(@Param('number') number: number) {
      return await this.pokemonService.findByNumber(number);
    }
  
    @Patch(':id')
    @ApiOperation({ summary: 'Atualizar um Pokémon' })
    @ApiParam({ name: 'id', description: 'ID do Pokémon' })
    @ApiResponse({ status: 200, description: 'Pokémon atualizado com sucesso.' })
    @ApiResponse({ status: 404, description: 'Pokémon não encontrado.' })
    async update(@Param('id') id: string, @Body() updatePokemonDto: UpdatePokemonDto) {
      return await this.pokemonService.update(id, updatePokemonDto);
    }
  
    @Delete(':id')
    @ApiOperation({ summary: 'Remover um Pokémon' })
    @ApiParam({ name: 'id', description: 'ID do Pokémon' })
    @ApiResponse({ status: 200, description: 'Pokémon removido com sucesso.' })
    @ApiResponse({ status: 404, description: 'Pokémon não encontrado.' })
    async remove(@Param('id') id: string) {
      return await this.pokemonService.remove(id);
    }
  
}

````

## 11. Configurando o módulo Pokemon
Edite o arquivo pokemon/pokemon.module.ts:
````
import { Module } from '@nestjs/common'; // Esta importação também parece estar faltando
import { PokemonController } from './pokemon.controller';
import { PokemonService } from './pokemon.service';
import { MongooseModule } from '@nestjs/mongoose';
import { Pokemon, PokemonSchema } from './schemas/pokemon.schema';
import { PokemonMongoRepository } from './repository/pokemon-repository.mongodb';
import { PokemonRepository } from './repository/pokemon-repository.interface';
import { POKEMON_REPOSITORY } from './pokemon.constants';

@Module({
  imports: [
    MongooseModule.forFeature([{ name: Pokemon.name, schema: PokemonSchema}])
  ],
  controllers: [PokemonController],
  providers: [
    PokemonService,
    {
      provide: POKEMON_REPOSITORY,
      useClass: PokemonMongoRepository,
    },
  ],
})
export class PokemonModule {}
````

## 12. Executando a aplicação
Agora inicie o MongoDB (certifique-se de que o MongoDB está instalado em sua máquina).
Em um terminal separado (Opcional):
````
mongod
````

Inicie a aplicação NestJS:
````
npm run start:dev
````

Acesse o Swagger para testar a API: http://localhost:3000/api


## Princípios SOLID aplicados:

### S - Single Responsibility Principle: Cada classe tem uma única responsabilidade.

- Controller lida com HTTP;

- Service lida com a lógica de negócios;

- Repository lida com o acesso aos dados;


### O - Open/Closed Principle: O código está aberto para extensão, fechado para modificação.

A interface do repositório permite adicionar novos métodos sem alterar o código existente.


### L - Liskov Substitution Principle: Usamos interfaces e classes que podem ser substituídas.

A implementação do repositório pode ser trocada sem afetar o restante do código.


### I - Interface Segregation Principle: Interfaces específicas para necessidades específicas.

Interface PokemonRepository define apenas métodos necessários.


### D - Dependency Inversion Principle: Dependências são injetadas e baseadas em abstrações.

Service depende da interface PokemonRepository, não da implementação concreta.



#### Este projeto apresenta uma API completa para gerenciar Pokémons com todos os padrões solicitados: DTOs, Service, Controller, Schema, Repository, e o enum de tipos.


## Estrutura de Pastas final (talvez)
````
pokemon-api/
├── node_modules/
├── src/
│   ├── main.ts                      # Ponto de entrada da aplicação com configuração do Swagger
│   ├── app.module.ts                # Módulo principal com conexão MongoDB
│   ├── app.controller.ts            # Controller principal (gerado pelo CLI)
│   ├── app.service.ts               # Service principal (gerado pelo CLI)
│   ├── pokemon/                     # Módulo Pokémon
│   │   ├── pokemon.module.ts        # Configuração do módulo
│   │   ├── pokemon.controller.ts    # Controller com endpoints da API
│   │   ├── pokemon.service.ts       # Service com lógica de negócios
│   │   ├── dto/                     # Data Transfer Objects
│   │   │   ├── create-pokemon.dto.ts # DTO para criação
│   │   │   └── update-pokemon.dto.ts # DTO para atualização
│   │   ├── enum/                    # Enumerações
│   │   │   └── pokemon-type.enum.ts # Tipos de Pokémon
│   │   ├── schemas/                 # Schemas do MongoDB
│   │   │   └── pokemon.schema.ts    # Schema do Pokémon
│   │   └── repository/              # Padrão Repository
│   │       ├── pokemon-repository.interface.ts  # Interface do repositório
│   │       └── pokemon-repository.mongodb.ts    # Implementação MongoDB
├── test/                            # Pasta para testes (gerada pelo CLI)
├── .eslintrc.js                     # Configuração do ESLint
├── .gitignore                       # Arquivos ignorados pelo Git
├── .prettierrc                      # Configuração do Prettier
├── nest-cli.json                    # Configuração do NestJS CLI
├── package.json                     # Dependências e scripts
├── package-lock.json                # Versões específicas das dependências
├── tsconfig.json                    # Configuração do TypeScript
├── tsconfig.build.json              # Configuração de build do TypeScript
└── README.md                        # Documentação do projeto
````

### Há muitos pontos que podemos melhorar, como a utilização da config do Swagger fora da main.ts e sim em uma pasta "Configs -> SwaggerConfig -> SwaggerConfig.ts", além disso a utilização de um .env para colocar informações como DB_URL, além desses exemplos existem muitos outros, espero que tenha te ajudado um pouco a entender como funciona a criação de uma Rest API com NestJs.

####  Para mais dúvidas, sugiro a utilização de IA para explicação de trechos de códigos e etc.
