---
title: Stack, Queue 알고리즘 3
description: 자바에서 제공하는 LinkedList로 구현한 알고리즘 문제
author: yoonxjoong
date: 2025-02-14 10:00:00 +0900
categories:
  - Algorithm
tags:
  - java
  - algorithm
---
개와 고양이만 분양하는 분양소가 있다  
분양받는 사람은 동물의 종류만 고를수 있고, 분양소에서 가장 오래 있는 순서로  
자동으로 분양될 동물이 정해지는 클래스를 구현하시오.  
단, 자바에서 제공하는 LinkedList로 구현하시오.


``` java
import java.util.LinkedList;  
  
enum AnimalType {  
    DOG, CAT  
}  
  
abstract class Animal {  
    AnimalType type;  
    String name;  
    int order;  
  
    Animal(AnimalType type, String name) {  
        this.type = type;  
        this.name = name;  
    }  
  
    int getOrder() {  
        return order;  
    }  
  
    void setOrder(int order) {  
        this.order = order;  
    }  
  
    String info() {  
        return order + ") type: " + type + ", name : " + name;  
    }  
}  
  
  
class Dog extends Animal {  
    Dog(String name) {  
        super(AnimalType.DOG, name);  
    }  
}  
  
class Cat extends Animal {  
    Cat(String name) {  
        super(AnimalType.CAT, name);  
    }  
}  
  
class AnimalShelter {  
    LinkedList<Animal> dogs = new LinkedList<>();  
    LinkedList<Animal> cats = new LinkedList<>();  
  
    int order;  
  
    AnimalShelter() {  
        order = 1;  
    }  
  
    void enqueue(Animal animal) {  
        animal.setOrder(order);  
        order += 1;  
  
        if (animal.type == AnimalType.DOG) {  
            dogs.addLast(animal);  
        } else if (animal.type == AnimalType.CAT) {  
            cats.addLast(animal);  
        }  
    }  
  
    Animal dequeueDog() {  
        return dogs.poll();  
    }  
  
    Animal dequeueCat() {  
        return cats.poll();  
    }  
  
    Animal dequeue() {  
        if (dogs.isEmpty() && cats.isEmpty()) {  
            return null;  
        } else if (dogs.isEmpty()) {  
            return cats.poll();  
        } else if (cats.isEmpty()) {  
            return dogs.poll();  
        }  
  
        Animal dog = dogs.peek();  
  
        Animal cat = cats.peek();  
  
        return dog.getOrder() > cat.getOrder() ? cats.poll() : dogs.poll();  
    }  
  
}  
  
public class test2 {  
    public static void main(String[] args) {  
        AnimalShelter animalShelter = new AnimalShelter();  
  
        Dog dog1 = new Dog("d1");  
        Dog dog2 = new Dog("d2");  
        Dog dog3 = new Dog("d3");  
        Dog dog4 = new Dog("d4");  
  
        Cat cat1 = new Cat("c1");  
        Cat cat2 = new Cat("c2");  
        Cat cat3 = new Cat("c3");  
        Cat cat4 = new Cat("c4");  
  
        animalShelter.enqueue(dog1);  
        animalShelter.enqueue(cat1);  
        animalShelter.enqueue(cat2);  
        animalShelter.enqueue(cat3);  
        animalShelter.enqueue(cat4);  
        animalShelter.enqueue(dog2);  
        animalShelter.enqueue(dog3);  
        animalShelter.enqueue(dog4);  
  
  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeue().info());  
        System.out.println(animalShelter.dequeueDog().info());  
        System.out.println(animalShelter.dequeueCat().info()); // NullPointException  
    }  
}
```
