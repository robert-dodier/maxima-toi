# Generating the table of integrals

The functions in this directory generate the table of integrals 
file `toi-integral-table.lisp`.  

It is generated with the command `make` in this directory.

Once generated it can be moved to the parent directory.  The generated file should be functionally equivalent to the file in the parent directory.  I have been not (yet) been able to eliminate all white space differences.

## Table entry

Each entry in the table of integrals is defined by a structure toi-entry.

<pre>
(defstruct toi-entry
  index       ; Number from source.  Used as index into lookup table
  integrand   ; integrand as string in maxima syntax
              ; not used if integrand2 defined
  comment     ; Optional comment
  var         ; variable of integration
  parameters  ; free parameters in integrand
              ; can be generated from integrand
  source      ; Source of entry
  integral    ; integral as string in maxima syntax
              ; not used if integral2 defined
  integrand2  ; expression to be integrated as lisp expression
              ; (can be generated from integrand)
  integral2   ; integral as lisp expression
              ; (can be generated from integral)
  lbound      ; lower limit of integration - nil if indefinite
  ubound      ; upper limit of integration - nil if indefinite
  constraint  ; predicate for constraint on parameters
  m2-pattern  ; m2 pattern matching expression for integrand
              ; Can be generated from integrand in simple cases
)
</pre>

## Preprocessor

A simple preprocessor - written in maxima and lisp - takes a minimal table entry and tries to fill in missing slots.

*  **index** is generated
*  **integrand2** can be generated from **integrand**
*  **integral2** can be generated from **integral**
*  **parameters** can be extracted from **integral2**
*  **m2-pattern** can be sometimes be generated from **integrand2** (for very simple cases).

The preprocessor functions are in file `toi-util.lisp`.

## A minimal entry

A minimal entry is 
<pre>
#S(TOI-ENTRY
   INTEGRAND "x*bessel_j(0,x)"
   SOURCE ("GR 5.56.2")
   INTEGRAL  "x*bessel_j(1,x)" )
</pre>

This is expanded to
<pre>
#S(TOI-ENTRY
  :INDEX 1112
  :INTEGRAND "x*bessel_j(0,x)"
  :VAR X
  :SOURCE ("GR 5.56.2")
  :INTEGRAL "x*bessel_j(1,x)"
  :INTEGRAND2 ((MTIMES) ((%BESSEL_J) 0 X) X)
  :INTEGRAL2   ((MTIMES) ((%BESSEL_J) 1 X) X)
  :M2-PATTERN ((MTIMES) ((%BESSEL_J) 0 (X VARP)) (X VARP)))
</pre>

## More complicated entries

A more complicated entry has:

* **parameters**
* a **constraint**, given by a predicate function of **VAR** and **PARAMETERS**

The following example has:

* Three **parameters** (A U V)
* A **constraint** that U+V#-1 or U+V+1#0.  This is noted in the comment and implemented as predicate `(NOT (ASKZEROSUMP U V 1))`

<pre>
#S(TOI-ENTRY
  :INDEX 1165
  :INTEGRAND "x^(u+v+1)*hankel_1(u,a*x)*hankel_1(v,a*x)"
  :COMMENT "u+v # -1"
  :VAR X
  :PARAMETERS (A U V)
  :SOURCE ("DLMF 10.22.7.i")
  :INTEGRAL 
"((hankel_1(u+1,a*x)*hankel_1(v+1,a*x)+hankel_1(u,a*x)*hankel_1(v,a*x))
 *x^(v+u+2)) /(2*(v+u+1))"
  :INTEGRAND2 ((MTIMES) ((%HANKEL_1) U ((MTIMES) A X)) ((%HANKEL_1) V ((MTIMES) A X)) ((MEXPT) X ((MPLUS) 1 U V)))
  :INTEGRAL2   
((MTIMES) ((RAT) 1 2) ((MEXPT) ((MPLUS) 1 U V) -1)
 ((MPLUS) ((MTIMES) ((%HANKEL_1) U ((MTIMES) A X)) ((%HANKEL_1) V ((MTIMES) A X)))
  ((MTIMES) ((%HANKEL_1) ((MPLUS) 1 U) ((MTIMES) A X)) ((%HANKEL_1) ((MPLUS) 1 V) ((MTIMES) A X))))
 ((MEXPT) X ((MPLUS) 2 U V)))
  :CONSTRAINT (NOT (ASKZEROSUMP U V 1))
  :M2-PATTERN 
((MTIMES) ((%HANKEL_1) (U FREEVAR) ((MTIMES) ((COEFFTT) (A FREEVAR)) (X VARP)))
 ((%HANKEL_1) (V FREEVAR) ((MTIMES) ((COEFFTT) (A_ EQUAL A)) (X VARP)))
 ((MEXPT) (X VARP) (U+V+1 SAMESAME U V 1))))
</pre>

## Bessel functions

Many integrals for Bessel J, Y and Hankel H1 and H2 functions 
have a common form.  See [DLMF 10.22](https://dlmf.nist.gov/10.22).  

This structure is used to used to generate entries from templates in `bessel-integral-template.lisp` using code in `bessel_c-integral-table.lisp`.  There are:

* four entries for integrals containing a single circular function
* ten entries for integrals containing the product of two circular functions, and
* additional special cases
