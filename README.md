


# Proyecto 2 - Compiladores

## Integrantes

- Adrian Sandoval Huamaní
- Andrés Jaffet Riveros Soto

## P1 (Typechecker/Codegen en IMP-FUN)


## P2 (Implementació de FCallStm)

### Parser

Ahora cuando lee una id, no solo se puede validad un `=`, sio también un `(` donde comienza el parseo de una función void tal como la gramática:

    Stm := ........
           id '(' CExp | (',' CExp)* ')'
           ........

```cpp
// in void parseStatement()
// ....
if (match(Token::ID)) {
  string lex = previous->lexema;
  if (match(Token::ASSIGN))
    s = new AssignStatement(lex, parseCExp());
  // new
  else if (match(Token::LPAREN)) {
    list<Exp*> args;
    if (!check(Token::RPAREN)) {
      args.push_back(parseCExp());
      while(match(Token::COMMA))
        args.push_back(parseCExp());
    }
    if (!match(Token::RPAREN)) parserError("Esperaba rparen");
    s = new FCallStatement(lex,args);
  } else {
    cout << "Error: esperaba = o lparen" << endl;
    exit(0);
  }
}
// ....
```

### AST

```cpp
// call stm: <id>(<args>);
class FCallStatement : public Stm {
public:
  string fname;
  list<Exp*> args;
  FCallStatement(string fname, list<Exp*> args);
  void accept(ImpVisitor* v);
  void accept(ImpValueVisitor* v);
  void accept(TypeVisitor* v);
  ~FCallStatement();
};
```

### Interpreter

```cpp
void ImpInterpreter::visit(FCallStatement* s) {
  FunDec* fdec = fdecs.lookup(s->fname);
  env.add_level();
  list<Exp*>::iterator it;
  list<string>::iterator varit;
  list<string>::iterator vartype;
  ImpVType tt;
  // comparar longitud
  if (fdec->vars.size() != s->args.size()) {
    cout << "Error: Numero de parametros incorrecto en llamada a " << fdec->fname << endl;
    exit(0);
  }
  for (it = s->args.begin(), varit = fdec->vars.begin(), vartype = fdec->types.begin();
       it != s->args.end(); ++it, ++varit, ++vartype) {
    tt = ImpValue::get_basic_type(*vartype);
    ImpValue v = (*it)->accept(this);
    if (v.type != tt) {
      cout << "Error FCall: Tipos de param y arg no coinciden. Funcion " << fdec->fname << " param " << *varit << endl;
      exit(0);
    }
    env.add_var(*varit, v);
  }
  retcall = false;
  fdec->body->accept(this);
  if (!retcall) {
    cout << "Error: Funcion " << s->fname << " no ejecuto RETURN" << endl;
    exit(0);
  }
  retcall = false;
  env.remove_level();
  return;
}
```

### Printer

```cpp
void ImpPrinter::visit(FCallStatement* s) {
  cout << s->fname << "(";
  list<Exp*>::iterator it;
  bool first = true;
  for (it = s->args.begin(); it != s->args.end(); ++it) {
    if (!first) cout << ", ";
    first = false;
    (*it)->accept(this);
  }
  cout << ")";
  return;
}
```

### TypeChecker

La implementación es similar al `FCallExp`, solo que ya no existe tipo de retorno `ImpType` así ya no interesa un valor a devolver.

```cpp
void ImpTypeChecker::visit(FCallStatement* e) {
  if (!env.check(e->fname)) {
    cout << "(Function call): " << "Funcion " << e->fname << " no definida" << endl;
    exit(0);
  }
  ImpType funtype = env.lookup(e->fname);
  int num_params = funtype.types.size()-1;
  int num_args = e->args.size();
  if (num_args != num_params) {
    cout << "Argumentos faltantes en la funcion " << e->fname << ", le espera " << num_params << ", pero pasa solo " << num_args << endl;
    exit(0);
  }
  ImpType argtype;
  list<Exp*>::iterator it;
  int i = 0;
  for (it = e->args.begin(); it != e->args.end(); ++it) {
    argtype = (*it)->accept(this);
    if (argtype.ttype != funtype.types[i]) {
      cout << "(Function call) Tipo de argumento no corresponde a tipo de parametro en fcall de: " << e->fname << endl;
      exit(0);
    }
    i++;
  }
  return;
}

```

### Codegen

```cpp
void ImpCodeGen::visit(FCallStatement* s) {
  FEntry fentry = analysis->ftable.lookup(s->fname);
  ImpType ftype = fentry.ftype;

  for (auto it = s->args.begin(); it != s->args.end(); ++it) {
    (*it)->accept(this);
  }
  codegen(nolabel,"pusha",get_flabel(s->fname));
  codegen(nolabel,"call");
  return;
}
```

## P3 (Impleemntación de ForDoStm)

### Parser

Se tiene los nuevos tokens: `FOR`, `IN` y `ENDFOR`. Ahora cuando identificar al inicio un "for", se parsea toda la estructura dada a la nueva expansión de la gramática:

    Stm := ........
           'for' id 'in' '(' CExp ',' CExp ')' 'do' Body 'endfor'
           ........


```cpp
// in void parseStatement()
// ....
else if (match(Token::FOR)) {
  if (!match(Token::ID))     parserError("Esperaba identifier");
  string id = previous->lexema;
  if (!match(Token::IN))     parserError("Esperaba 'in'");
  if (!match(Token::LPAREN)) parserError("Esperaba 'lparen'");
  Exp* e1 = parseCExp();
  if (!match(Token::COMMA))  parserError("Esperaba comma");
  Exp* e2 = parseCExp();
  if (!match(Token::RPAREN)) parserError("Esperaba 'rparen'");
  if (!match(Token::DO))     parserError("Esperaba 'do'");
  tb = parseBody();
  if (!match(Token::ENDFOR)) parserError("Esperaba 'endfor'");
  s = new ForDoStatement(id,e1,e2,tb);
}
// ....
```

### AST

```cpp
// for <id> in (<start>, <end>) do <body> endfor
class ForDoStatement : public Stm {
public:
  string id;
  Exp* start, *end;
  Body *body;
  ForDoStatement(string id, Exp* start, Exp* end, Body* b);
  void accept(ImpVisitor* v);
  void accept(ImpValueVisitor* v);
  void accept(TypeVisitor* v);
  ~ForDoStatement();
};
```

### Interpreter

```cpp
void ImpInterpreter::visit(ForDoStatement* s) {
  ImpValue start = s->start->accept(this);
  ImpValue end = s->end->accept(this);
  if (start.type != TINT || end.type != TINT) {
    cout << "Error: Tipos de start y end en el for deben ser enteros" << endl;
    exit(0);
  }
  env.add_level();
  env.add_var(s->id, start);
  ImpValue v;
  v.set_default_value(TINT);
  if (start.int_value > end.int_value) {
    cout << "Warning: start > end en for" << endl;
  }
  for (int i = start.int_value; i <= end.int_value; i++) {
    v.int_value = i;
    env.update(s->id, v);
    s->body->accept(this);
  }
  env.remove_level();
  return;
}
```

### Printer

```cpp
void ImpPrinter::visit(ForDoStatement* s) {
  cout << "for " << s->id << " in (";
  s->start->accept(this);
  cout << ", ";
  s->end->accept(this);
  cout << ") do" << endl;
  s->body->accept(this);
  cout << "endfor";
  return;
}
```

### TypeChecker

```cpp
void ImpTypeChecker::visit(ForDoStatement* s) {
  ImpType type = s->start->accept(this);
  if (!type.match(inttype)) {
    cout << "Tipo de start en ForDoStatement debe de ser int" << endl;
    exit(0);
  }
  type = s->end->accept(this);
  if (!type.match(inttype)) {
    cout << "Tipo de end en ForDoStatement debe de ser int" << endl;
    exit(0);
  }
  env.add_var(s->id, inttype);
  int sp_before = sp;
  s->body->accept(this);
  sp = sp_before;
  return;
}
```

### Codegen

```cpp
void ImpCodeGen::visit(ForDoStatement* s) {
  string l1 = next_label();
  string l2 = next_label();
  string l3 = next_label();
  string l4 = next_label();
  current_dir++;
  VarEntry ventry;
  ventry.dir = current_dir;
  ventry.is_global = false;
  direcciones.add_var(s->id, ventry);
  s->start->accept(this);
  codegen(nolabel,"store",ventry.dir);
  codegen(l1,"skip");
  codegen(nolabel,"load",ventry.dir);
  s->end->accept(this);
  codegen(nolabel,"sub");
  codegen(nolabel,"jmpz",l2);
  s->body->accept(this);
  codegen(nolabel,"load",ventry.dir);
  codegen(nolabel,"push",1);
  codegen(nolabel,"add");
  codegen(nolabel,"store",ventry.dir);
  codegen(nolabel,"goto",l1);
  codegen(l2,"skip");
  return;
}
```