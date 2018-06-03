# The Definitive ANTLR Reference

## Chapter 4

### Matching an Arithmetic Expression Language


#### Expr Grammar
```ANTLR
grammar Expr;

prog: stat+ ;

stat: expr NEWLINE
  | ID '=' expr NEWLINE
  | NEWLINE
  ;

expr: expr ('*'|'/') expr
  | expr ('+'|'-') expr
  | INT
  | ID
  | '(' expr ')'
  ;

ID : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE: 'r'?'\n' ;
WS : [ \t]+ -> skip ;
```