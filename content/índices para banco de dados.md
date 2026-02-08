# (django) índices de banco de dados
Os `Index` no Django são usados para criar índices de banco de dados diretamente nos modelos, melhorando a performance das consultas. Em Django, a estrutura de dados padrão que serve de índice sobre uma tabela do banco de dados é uma _árvore binária_ (B-Tree).

A maioria dos bancos de dados oferece alguma _tecnologia de indexação_ para pesquisa em tempo _sub-linear._

Índices podem ser criados a partir da classe Meta, dentro de uma classe que é um modelo de dados (isto é, uma subclasse de `django.db.models.Model`, e que representa uma tabela no banco de dados):

```python
from django.db import models

class DataModel(models.Model):
	class Meta:
		# índices são definidos aqui
		indexes = [ ... ]
	
...
```

Esta é a forma mais apropriada e que permite uma variada gama de tecnologias de indexação. Para índices simples, sobre somente um campo individual, podemos fazer:

```python
class Produto(models.Model):
	# índice simples
    codigo = models.CharField(max_length=50, db_index=True)
    nome = models.CharField(max_length=100)
```

É possível indexar uma tabela de diversos modos. Abaixo são expostas algumas maneiras.

## índice _B-Tree_ padrão

```python
from django.db import models

class Produto(models.Model):
    nome = models.CharField(max_length=100)
    preco = models.DecimalField(max_digits=10, decimal_places=2)
    categoria = models.CharField(max_length=50)
    
    class Meta:
        indexes = [
            models.Index(fields=['nome']),  # Índice em um campo
            models.Index(fields=['categoria', 'preco']),  # Índice composto
        ]
```

---
## índices funcionais
O argumento posicional `*expressions` permite indexação baseada em expressões e funções de banco de dados. As funções para banco de dados, em Django, são encontradas no módulo `django.db.models.functions`.

```python
from django.db.models import Index
from django.db.models.functions import Upper, Lower

class Cliente(models.Model):
    nome = models.CharField(max_length=100)
    email = models.EmailField()
    
    class Meta:
        indexes = [
            Index(Upper('nome'), name='idx_nome_upper'),
            Index(Lower('email'), name='idx_email_lower'),
        ]
```

- Índices funcionais foram introduzidos no Django `3.2`.

---
## indexação com ordenação
```python
from django.db.models import Index

class Pedido(models.Model):
    numero = models.CharField(max_length=20)
    data = models.DateTimeField()
    valor_total = models.DecimalField(max_digits=10, decimal_places=2)
    
    class Meta:
        indexes = [
            Index(fields=['-data']),  # Ordem descendente
            Index(fields=['numero', '-data']),  # Misto
        ]
```
Note que:
1. `'data'` seria ordenar todos os objetos `Pedido` por seu campo `data`, _em ordem ascendente_, normal. Agora `'-data'` é o contrário disto, isto é, ordenar todos os objetos `Pedido` por seu campo `data`, *em ordem descendente.*
2. `Index(fields=['numero', '-data']` é ordenar todos os `Pedido`s por seu `numero`, em ordem ascendente, e por sua `data`, em ordem descendente.

Mais um exemplo interessante de índice composto com ordenação:

```python
class Loja(models.Model):
    nome = models.CharField(max_length=100)
    cidade = models.CharField(max_length=50)
    faturamento = models.DecimalField(max_digits=15, decimal_places=2)
    
    class Meta:
        indexes = [
            models.Index(
                fields=['cidade', '-faturamento'],
                name='idx_cidade_faturamento_desc'
            ),
        ]
```

---
## indexação parcial (com condição `WHERE`)
### primeiro exemplo
```python
from django.db.models import Q, Index

class Produto(models.Model):
    nome = models.CharField(max_length=100)
    preco = models.DecimalField(max_digits=10, decimal_places=2)
    ativo = models.BooleanField(default=True)
    
    class Meta:
        indexes = [
            Index(
                fields=['nome'],
                condition=Q(ativo=True),
                name='idx_nome_ativos'
            ),
            Index(
                fields=['preco'],
                condition=Q(preco__gt=100),
                name='idx_precos_altos'
            ),
        ]
```

### segundo exemplo
```python
from django.db.models import F

class Funcionario(models.Model):
    nome = models.CharField(max_length=100)
    salario = models.DecimalField(max_digits=10, decimal_places=2)
    departamento = models.CharField(max_length=50)
    
    class Meta:
        indexes = [
            Index(
                expressions=[F('departamento'), F('salario').desc()],
                condition=Q(salario__gt=5000),
                name='idx_depto_salario_alto'
            ),
        ]
```



---
## indexação para PostgreSQL
As técnicas expostas abaixo esperam PostgreSQL 11+.

### _index covering_
```python
class Venda(models.Model):
    data = models.DateField()
    cliente_id = models.IntegerField()
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20)
    
    class Meta:
        indexes = [
            Index(
                fields=['data'],
                include=['cliente_id', 'total'],
                name='idx_data_covering'
            ),
        ]
```

### _Opclasses_
```python
from django.contrib.postgres.indexes import OpClass

class Artigo(models.Model):
    titulo = models.CharField(max_length=200)
    conteudo = models.TextField()
    data_publicacao = models.DateField()
    
    class Meta:
        indexes = [
            Index(
                fields=['titulo'],
                opclasses=['varchar_pattern_ops'],
                name='idx_titulo_pattern'
            ),
        ]
```

---
# boas práticas
1. **Use índices para campos frequentemente filtrados**    
2. **Índices compostos devem seguir a ordem das consultas**    
3. **Evite muitos índices em tabelas com muitas escritas**    
4. **Use nomes descritivos para índices**    
5. **Considere índices parciais para dados filtrados**    
6. **Teste performance antes e depois de criar índices**

---
# verifique os índices do seu banco de dados
```sql
-- PostgreSQL
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'app_modelo';

-- SQLite
PRAGMA index_list('app_modelo');
```

---
# referências
1. [Database index @ Wikipedia](https://en.wikipedia.org/wiki/Database_index)

---
