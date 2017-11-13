# Physics Engine

*** Overview ***

The goal of this project is to develop a real-time 2D physics engine. 

This physics engine consists of 6 major parts:
1. Vector and matrix library 
2. Particle physics module 
3. Rigid body physics module 
4. Force generation module 
5. Collision detection module 
6. Collision response module 
7. Integration of simulation modules 

*** Module-level Descriptions and Design ***

*1. Vector and Matrix Library 
Given an input of vectors and matrixes, the module produces the result for a desired operation. This module provides the foundation for all simulation engine calculations.

There are 3 Mathematical primitives: 
2-dimensional vector
2x2 matrix
2x3 matrix

The following operations can be performed on the 3 primitives:
Vector addition, subtraction, scaling
Vector dot product, cross product
Vector rotation
Matrix determinant
Matrix inverse, transpose
Matrix transformation of vector

Excerpt from “core.cpp” for vector dot product
return x * v.x + y * v.y;

Excerpt from “core.cpp” for vector rotation
	real x2 = x * cos(angle) - y * sin(angle);
	real y2 = x * sin(angle) + y * cos(angle);

Excerpt from “core.cpp” for matrix determinant
return (data[0] * data[3] - data[1] * data[2]);

For complete header file, see appendix D.

There are many ways to represent a matrix, I decided to use row-major order in compliance with OpenGL matrix.  I used homogeneous coordinate system because homogeneous coordinates allow us to combine translation and rotation into a single transformation matrix, thus saving computer memory.

*2. Particle Physics Module
The particle is an infinitesimally small point mass that reacts to an applied force. Each particle needs to keep track of its own position, velocity, acceleration, mass, and the accumulated force in every frame of the simulation. Newton’s second law gives us the resulting acceleration from force, and the acceleration can be integrated to get the updated position.

The module takes the input force generated by the force generation module and outputs the updated position, velocity, acceleration. 

Excerpt from “particle.h” for private variables of the Particle class
Vector2 position;
	Vector2 velocity;
	Vector2 acceleration;
	real inverseMass; // 1/m
	real damping; // 0 to 1
	Vector2 forceAccum;

Excerpt from “particle.cpp” for position integration
	acceleration = forceAccum * inverseMass;
	velocity.addScaledVector(acceleration, duration);
position.addScaledVector(velocity, duration);

We can break down complicated shapes into a large number of interconnected particles. This is a mass-aggrevate model. However, large number of particles require more memory and have slower performance. The alternative is to simulate shapes with rigid bodies. The user has the freedom to choose which model to use.



*3. Rigid Body Physics Module 
A rigid body is a mass with finite shape and size that has a definite orientation. In addition to all the functionalities of a particle, a rigid body must also keep track of its own orientation, angular velocity, angular acceleration, accumulated torque, and moment of inertia.

The module takes the input force generated by the force generation module and outputs the updated position, velocity, acceleration, orientation, angular velocity, angular acceleration. 

Excerpt from “body.h” for private variables of the RigidBody class
Vector2 position; // position of center of mass
	Vector2 orientation;
	Vector2 velocity;
	real angularVelocity;
	Vector2 acceleration;
	Matrix3 transformMatrix; // only rotation and translation
	real inverseMass;
	real inverseMomentOfInertia;
	real linearDamping; // 0 to 1
	real angularDamping; // 0 to 1
	Vector2 forceAccum;
	real torqueAccum;

Excerpt from “body.cpp” for position and orientation integration
	Vector2 a = acceleration;
	a.addScaledVector(forceAccum, inverseMass);
	real angularAcceleration = torqueAccum * inverseMomentOfInertia;
	velocity.addScaledVector(a, duration);
	angularVelocity += angularAcceleration * duration;
	position.addScaledVector(velocity, duration);
	orientation.rotate(angularVelocity * duration);


The position of the rigid body is chosen to be placed at the center of mass. This makes the physics calculation much easier, since forces applied at the center of mass generates zero torque.


*4. Force Generation Module 
The force generation module generates the forces that are applied to the particles and rigid bodies. This is where gravity, air drag, and buoyancy are generated. The module takes the current system state data and outputs the generated forces.

The types of forces for rigid bodies include:
Gravitational force
Spring force
Radial magnetic force
Aerodynamic drag force
Buoyancy force

Excerpt from “fgen.cpp” for damped spring force generation using Hook’s Law:
Vector2 deltaV = body->getVelocityAtPoint(connectionPoint)
		- other->getVelocityAtPoint(otherConnectionPoint);
	// F = -k * (l - l0) - c * v
	Vector2 force = l.unit() * (-(l.magnitude() - restLength) * springConstant)
		- deltaV * dampingCoefficient;
	body->addForceAtPoint(force, connectionWorld);

Here, the damped spring force generator needs to monitor the position of connected rigid bodies and their velocities to compute the force required.

Custom forces can be easily modified and defined. Note that collision forces are not generated here, but handled in the collision subsystem instead.


*5. Collision Detection Module 
The collision detection module detects intersection between collision primitives and offsets of constraints. Contacts of constraints are internal collisions caused by joints and links. There are 3 collision primitives: spheres (circles), boxes (rectangles), and planes (surfaces). Spheres and boxes can be attached to a rigid body. Multiple spheres and boxes can be attached to a single rigid body to create a complex shape. Planes are stationary and are not attached to rigid bodies. There are 6 different sets of collision between each primitive. Each of them are handled by a different algorithm. Here is a brief description of these algorithms.

Table 5-1: Table of collision primitive algorithms

Primitive Primitive Collision detection algorithm
sphere sphere Check the distance between the centers of the spheres
sphere plane Check the distance between the center of sphere and plane
box plane Check the intersection of each vertex of box with plane
box sphere Clamp the location of center of sphere onto the side of the box, and check the distance of the clamped location to the center of the sphere
box box Separating axis theorem - if there exist an axis that separates the two boxes, they are not intersecting

This module gets the input from system states data, checks for collisions and return the corresponding contact data. The contact data includes:
Rigid bodies involved in the collision
Point of contact
Contact normal vector (the direction of contact)
Depth of penetration
Contact friction
Coefficient of restitution

*6. Collision Response Module 
The collision response module takes all the contacts detected in a frame, apply the collision impulses, and resolve the interpenetrations.

Given the contact data, we can compute the desired final velocity of contacting rigid bodies. The coefficient of restitution scales the velocity along the contact normal vector, while the friction scales the velocity perpendicular to the contact normal. Knowing the desired final velocity, we can compute the amount of impulse needed to apply at the point of contact using laws of collision physics.

The simulation is run in discrete time steps or frames. Thus, interpenetrations of the collision primitives are inevitable. When interpenetrations occur, the module moves the primitives apart with a balance of translation of rotation. 


Figure 6-1: Before and after collision interpenetration resolution of two boxes

Here is the pseudo-code for the collision response algorithm:
Set up all the contacts
Iterate through the contacts
{
Compute the desired final velocity
Compute the required impulse
Apply the impulse
}
Iterate through the contacts
	Translate and rotate the bodies apart from interpenetration


The physics engine successfully simulated designs such as Newton’s cradle, piston, and rope bridge. Thus, it is able to accurately simulate and display the required number of rigid bodies complying with laws of physics in real time. 
