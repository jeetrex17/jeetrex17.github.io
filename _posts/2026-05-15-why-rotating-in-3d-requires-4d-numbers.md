---
title: "Why Rotating in 3D Requires Numbers That Live in 4D"
date: 2026-05-15
categories: [Robotics, Math]
tags: [robotics, linear-algebra, quaternions, drone, complex-numbers]
layout: post
math: true
description: "Climbing the number ladder from real numbers to complex numbers to quaternions, and why 3D rotation needs a 4D number system."
---

Last post I waved my hands and said quaternions are 4 numbers and you should just trust me on it. So in this post we are going to actually derive them from scratch. By the end of this you will understand **why** we use a 4D number system to do 3D rotation, not just how.

Heads up: this post has more math than the last one. I had to read this stuff 3 times before it clicked for me, so take it slow, and if anything is unclear pls ping me.

> A quick reminder on coordinate frames from the [last post](/posts/what-does-forward-even-mean-to-a-drone/), the drone has its own body frame that tilts with it, and the world frame stays fixed. Quaternions are the thing we use to keep track of "how is the body frame oriented relative to the world frame right now". If that sentence makes sense you are good to keep reading.
{: .prompt-info }

## The Ladder of Numbers

The cleanest way I have seen to understand quaternions is to climb up a ladder. Each rung of the ladder is a number system, and each one solves a rotation problem that the rung below cannot solve. Let's walk up.

### Rung 1: Real Numbers (1D)

A real number is just a point on a line:

$$
x \in \mathbb{R}
$$

You can flip the sign, $x \to -x$, and you could maybe call that a "rotation" by $180°$. But thats the only kind of rotation a 1D number can express. You cannot represent a $45°$ rotation on a line because a line has no concept of angle. So real numbers are useless for general rotation.

### Rung 2: Complex Numbers (2D)

A complex number is a 2D number:

$$
z = a + bi, \quad \text{where } i^2 = -1
$$

If you plot it on a plane, $a$ goes on the real axis and $b$ goes on the imaginary axis. So a complex number is literally a point in 2D.

Now here is the magic: **multiplying by $e^{i\theta}$ rotates a point by angle $\theta$ in the plane**. Using Euler's identity:

$$
e^{i\theta} = \cos\theta + i\sin\theta
$$

So if you have a 2D point $z$ and you want to rotate it by $\theta$, you just do:

$$
z' = e^{i\theta} \cdot z
$$

That's it. One multiplication. Compare that to the rotation matrix way:

$$
\begin{bmatrix} x' \\ y' \end{bmatrix} =
\begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix}
\begin{bmatrix} x \\ y \end{bmatrix}
$$

Same thing, uglier notation. The complex number form is so clean it almost feels like cheating. The takeaway is: **complex numbers are the natural number system for 2D rotation**.

### The Obvious Question: What About 3D?

If 1D rotation needs $\mathbb{R}$ (sort of) and 2D rotation needs $\mathbb{C}$, then by the same pattern 3D rotation should need some 3-dimensional number system. Like a triplet $(a, b, c)$ with maybe two imaginary units $i$ and $j$, and a rule for how they multiply.

People tried for years to make this work and it just does not. The deep reason is that if you want multiplication to behave nicely (closure, associativity, an inverse for every nonzero element), there is no 3D version that works like the real numbers or complex numbers.

The fix turns out to be weird and beautiful: add **one more dimension**. Move to 4D. And then it all works.

### Rung 3: Quaternions (4D)

A quaternion is a 4D number:

$$
\mathbf{q} = w + xi + yj + zk, \quad w, x, y, z \in \mathbb{R}
$$

Three imaginary units instead of one, with the multiplication rule:

$$
i^2 = j^2 = k^2 = ijk = -1
$$

From that one line you can derive the full multiplication table:

$$
ij = k, \quad jk = i, \quad ki = j
$$
$$
ji = -k, \quad kj = -i, \quad ik = -j
$$

The key thing to notice: **$ij = k$ but $ji = -k$**. The order matters. Quaternion multiplication is **not commutative**. This is what makes 3D rotation work, because 3D rotations themselves are not commutative either (try rotating your phone by $90°$ around two different axes in different orders, you get different results).

We usually split a quaternion into a scalar part and a vector part:

$$
\mathbf{q} = (w, \mathbf{v}), \quad w \in \mathbb{R}, \quad \mathbf{v} = (x, y, z) \in \mathbb{R}^3
$$

This is the form you will see in code most often. In my simulator the state stores it as four separate floats `qw, qx, qy, qz`.

## Unit Quaternions and Axis-Angle

### Why Unit Quaternions

Just like only the unit complex numbers $\lvert z \rvert = 1$ correspond to pure rotation (anything else would also scale), only **unit quaternions** correspond to pure rotation. A unit quaternion satisfies:

$$
\|\mathbf{q}\|^2 = w^2 + x^2 + y^2 + z^2 = 1
$$

Geometrically the set of unit quaternions forms the surface of a 4D hypersphere, called $S^3$. Every point on $S^3$ is a valid 3D rotation. Mind bending but true.

One subtle thing worth flagging now: the mapping from unit quaternions to actual rotations is **two-to-one**. The quaternions $\mathbf{q}$ and $-\mathbf{q}$ describe the **exact same rotation**. You can see why from the axis-angle form, flipping every sign is the same as $(\theta + 2\pi)$ around the same axis, which is geometrically the same orientation. This is called the **double cover** of the rotation group $SO(3)$ by $S^3$. It will come back to bite us in the slerp section where we have to pick the "shorter" of two paths.

### The Axis-Angle Encoding

Here is how a unit quaternion actually represents a rotation. Pick a unit vector $\hat{\mathbf{n}} = (n_x, n_y, n_z)$ as your axis and an angle $\theta$, then:

$$
\mathbf{q} = \cos\frac{\theta}{2} + \sin\frac{\theta}{2}\,(n_x i + n_y j + n_z k)
$$

Or in scalar-vector form:

$$
\mathbf{q} = \left(\cos\frac{\theta}{2}, \, \sin\frac{\theta}{2}\,\hat{\mathbf{n}}\right)
$$

**Why the half-angle?** This is the question I kept asking myself. The half is going to fall out naturally when we do the sandwich product in the next section, so for now just accept it and we will earn the intuition shortly.

### Identity Quaternion

Plug in $\theta = 0$ (no rotation):

$$
\mathbf{q} = (\cos 0, \sin 0 \cdot \hat{\mathbf{n}}) = (1, 0, 0, 0)
$$

This is the **identity quaternion**. And remember from the last post our simulator's initial state:

```cpp
state_ = {0,0,1, 0,0,0, 1,0,0,0, 0,0,0};
//        px py pz vx vy vz qw qx qy qz wx wy wz
```

That `1, 0, 0, 0` in the middle is exactly this. The drone starts with zero rotation.

## Quaternion Operations

### Multiplication (Hamilton Product)

The product of two quaternions $\mathbf{q}_1 = (w_1, \mathbf{v}_1)$ and $\mathbf{q}_2 = (w_2, \mathbf{v}_2)$ is:

$$
\mathbf{q}_1 \mathbf{q}_2 = \left(w_1 w_2 - \mathbf{v}_1 \cdot \mathbf{v}_2, \;\; w_1 \mathbf{v}_2 + w_2 \mathbf{v}_1 + \mathbf{v}_1 \times \mathbf{v}_2\right)
$$

The cross product showing up here is exactly why quaternion multiplication is non-commutative, because $\mathbf{v}_1 \times \mathbf{v}_2 = -\mathbf{v}_2 \times \mathbf{v}_1$.

When we compose two rotations using $\mathbf{q}_1 \mathbf{q}_2$, the convention is **"apply $\mathbf{q}_2$ first, then $\mathbf{q}_1$"**, just like matrix multiplication.

In code, the helper looks something like:

```cpp
Quaternion qmul(const Quaternion& a, const Quaternion& b) {
    return {
        a.w*b.w - a.x*b.x - a.y*b.y - a.z*b.z,
        a.w*b.x + a.x*b.w + a.y*b.z - a.z*b.y,
        a.w*b.y - a.x*b.z + a.y*b.w + a.z*b.x,
        a.w*b.z + a.x*b.y - a.y*b.x + a.z*b.w
    };
}
```

16 scalar multiplications and 12 add/subtracts in this straight expansion. There are clever ways to rearrange the arithmetic, but this direct form is what you actually see in most code, and it is cheap enough on modern hardware that nobody bothers.

### Conjugate and Inverse

The conjugate flips the sign of the vector part:

$$
\mathbf{q}^{\ast} = (w, -\mathbf{v}) = w - xi - yj - zk
$$

And for **unit quaternions**:

$$
\mathbf{q}^{-1} = \mathbf{q}^{\ast}
$$

Quick proof, compute $\mathbf{q} \mathbf{q}^{\ast}$ using the Hamilton product:

$$
\mathbf{q} \mathbf{q}^{\ast} = (w \cdot w - \mathbf{v} \cdot (-\mathbf{v}), \, \ldots) = (w^2 + \|\mathbf{v}\|^2, \, \mathbf{0}) = (1, \mathbf{0})
$$

The vector terms all cancel and the scalar is $\|\mathbf{q}\|^2 = 1$. So $\mathbf{q} \mathbf{q}^{\ast} = 1$, meaning $\mathbf{q}^{\ast}$ is the inverse.

This is the **exact reason** for the trick from the last post:

```cpp
qd_quat_rot(state_.qw, -state_.qx, -state_.qy, -state_.qz,
            0.0, 0.0, QD_GRAVITY, ax, ay, az);
```

The negative signs are constructing $\mathbf{q}^{\ast}$ on the fly. The orientation quaternion rotates body to world, so its conjugate (which is its inverse for unit quaternions) rotates world to body. That's how we get gravity into body frame for the simulated accelerometer.

## Rotating a Vector: The Sandwich

This is the part that finally makes the half-angle thing make sense.

### Setup

Take a 3D vector $\mathbf{v} \in \mathbb{R}^3$ and embed it as a **pure quaternion** (zero scalar part):

$$
\mathbf{v}_q = (0, \mathbf{v})
$$

The claim is: to rotate $\mathbf{v}$ by quaternion $\mathbf{q}$, compute:

$$
\mathbf{v}'_q = \mathbf{q} \, \mathbf{v}_q \, \mathbf{q}^{\ast}
$$

The result is another pure quaternion, and its vector part is the rotated $\mathbf{v}'$.

### The Derivation

Let's actually grind through it, because the half-angle pops out and it is satisfying.

Substitute the axis-angle form:

$$
\mathbf{q} = c + s\,\hat{\mathbf{n}}, \quad \text{where } c = \cos\tfrac{\theta}{2}, \; s = \sin\tfrac{\theta}{2}
$$

And treat $\hat{\mathbf{n}}$ as a pure quaternion $(0, \hat{\mathbf{n}})$. So $\mathbf{q} = (c, s\hat{\mathbf{n}})$ and $\mathbf{q}^{\ast} = (c, -s\hat{\mathbf{n}})$.

**Step 1.** Compute $\mathbf{q} \mathbf{v}_q$ using the Hamilton product:

$$
\mathbf{q} \mathbf{v}_q = (c, s\hat{\mathbf{n}}) \cdot (0, \mathbf{v}) = (-s\,\hat{\mathbf{n}} \cdot \mathbf{v}, \; c\mathbf{v} + s\,\hat{\mathbf{n}} \times \mathbf{v})
$$

**Step 2.** Multiply the result by $\mathbf{q}^{\ast}$ on the right:

$$
(\mathbf{q} \mathbf{v}_q)\,\mathbf{q}^{\ast} = \big(-s\,\hat{\mathbf{n}} \cdot \mathbf{v}, \; c\mathbf{v} + s\,\hat{\mathbf{n}} \times \mathbf{v}\big) \cdot (c, -s\hat{\mathbf{n}})
$$

Apply the product rule again. The scalar part of the result is:

$$
-s c\,(\hat{\mathbf{n}} \cdot \mathbf{v}) - (c\mathbf{v} + s\,\hat{\mathbf{n}} \times \mathbf{v}) \cdot (-s\hat{\mathbf{n}}) = -s c\,(\hat{\mathbf{n}} \cdot \mathbf{v}) + s c\,(\hat{\mathbf{n}} \cdot \mathbf{v}) + s^2 (\hat{\mathbf{n}} \times \mathbf{v}) \cdot \hat{\mathbf{n}}
$$

The first two terms cancel and $(\hat{\mathbf{n}} \times \mathbf{v}) \cdot \hat{\mathbf{n}} = 0$, so the scalar part is zero. Good, the result is a pure quaternion as expected.

**Step 3.** The vector part is the rotated $\mathbf{v}'$. Working it out (skipping a few lines of cross product algebra):

$$
\mathbf{v}' = (c^2 - s^2)\mathbf{v} + 2cs\,(\hat{\mathbf{n}} \times \mathbf{v}) + 2s^2\,(\hat{\mathbf{n}} \cdot \mathbf{v})\,\hat{\mathbf{n}}
$$

**Step 4.** Now use the double angle trig identities:

$$
c^2 - s^2 = \cos^2\tfrac{\theta}{2} - \sin^2\tfrac{\theta}{2} = \cos\theta
$$
$$
2cs = 2\sin\tfrac{\theta}{2}\cos\tfrac{\theta}{2} = \sin\theta
$$
$$
2s^2 = 2\sin^2\tfrac{\theta}{2} = 1 - \cos\theta
$$

Plug them in:

$$
\boxed{\;\mathbf{v}' = \cos\theta\,\mathbf{v} + \sin\theta\,(\hat{\mathbf{n}} \times \mathbf{v}) + (1 - \cos\theta)(\hat{\mathbf{n}} \cdot \mathbf{v})\,\hat{\mathbf{n}}\;}
$$

This is **Rodrigues' rotation formula**. It rotates the vector $\mathbf{v}$ by angle $\theta$ around the axis $\hat{\mathbf{n}}$, and we just derived it purely from quaternion algebra. Pretty cool.

### Why the Half Angle, Finally

Look at what just happened. We started with a quaternion encoding $\theta/2$. Both factors in the sandwich, $\mathbf{q}$ on the left and its conjugate $\mathbf{q}^{\ast}$ on the right, carry that same $\theta/2$ angle (with $\mathbf{q}^{\ast}$ flipping the sign of the vector part). When you grind through the Hamilton products, every term ends up as a product or square of $\sin\frac{\theta}{2}$ and $\cos\frac{\theta}{2}$. The double angle identities then collapse those into $\sin\theta$ and $\cos\theta$ in the final formula.

So it is not that we "rotate twice". $\mathbf{q}^{\ast}$ is the inverse of $\mathbf{q}$, applying it on its own would undo the rotation, and the conjugate on the right is structurally needed to make the scalar part of the result vanish (so we get back a clean pure-quaternion vector). The half-angle is a consequence of the algebra, not of a double application of the same rotation. If you put the full angle $\theta$ into $\mathbf{q}$ instead of $\theta/2$, the same derivation would spit out a rotation by $2\theta$, because the trig identities do what they do regardless of which angle you feed in. So we pre-cancel by encoding $\theta/2$

This bugged me for a while until I sat down and did this derivation. Once you see the trig identities chew through the products, it stops being magic.
  
## Integrating Angular Velocity (Gyro to Orientation)

So far we have static rotations. But our drone is constantly rotating, the gyroscope on the IMU is streaming an angular velocity vector $\boldsymbol{\omega} = (\omega_x, \omega_y, \omega_z)$ in body frame. We need to integrate that into orientation, update $\mathbf{q}$ at every timestep.

### Deriving the Quaternion Derivative

Treat $\boldsymbol{\omega}$ as a pure quaternion:

$$
\boldsymbol{\omega}_q = (0, \omega_x, \omega_y, \omega_z)
$$

A small rotation by $\boldsymbol{\omega}\,dt$ has axis $\hat{\boldsymbol{\omega}} = \boldsymbol{\omega} / \|\boldsymbol{\omega}\|$ and angle $\|\boldsymbol{\omega}\|\,dt$. The corresponding quaternion is:

$$
\delta\mathbf{q} = \left(\cos\tfrac{\|\boldsymbol{\omega}\|\,dt}{2}, \, \sin\tfrac{\|\boldsymbol{\omega}\|\,dt}{2}\,\hat{\boldsymbol{\omega}}\right)
$$

For very small $dt$, the cosine is approximately $1$ and the sine is approximately its argument:

$$
\delta\mathbf{q} \approx \left(1, \tfrac{1}{2}\,\boldsymbol{\omega}\,dt\right) = (1, 0, 0, 0) + \tfrac{1}{2}\,\boldsymbol{\omega}_q\,dt
$$

Now the updated orientation is $\mathbf{q}(t + dt) = \mathbf{q}(t) \cdot \delta\mathbf{q}$ (applying the small rotation in the body frame). Substituting:

$$
\mathbf{q}(t + dt) = \mathbf{q}(t) + \tfrac{1}{2}\,\mathbf{q}(t)\,\boldsymbol{\omega}_q\,dt
$$

Rearranging:

$$
\boxed{\;\dot{\mathbf{q}} = \frac{1}{2}\,\mathbf{q}\,\boldsymbol{\omega}_q\;}
$$

This is the **quaternion kinematic equation**. The $\frac{1}{2}$ comes from the same half-angle source we kept seeing earlier.

### The RK4 Step in Code

In the simulator, this equation gets fed into RK4 (fourth order Runge-Kutta integration). Schematically:

```cpp
QuadState qd_rk4(const QuadState& s, const std::array<double,4>& motors, double dt) {
    auto k1 = qd_derivative(s, motors);
    auto k2 = qd_derivative(s + 0.5*dt*k1, motors);
    auto k3 = qd_derivative(s + 0.5*dt*k2, motors);
    auto k4 = qd_derivative(s + dt*k3, motors);
    return s + (dt/6.0)*(k1 + 2*k2 + 2*k3 + k4);
}
```

And inside `qd_derivative`, for the quaternion part:

```cpp
// dq/dt = 0.5 * q * omega
Quaternion dq;
dq.w = 0.5 * (-s.qx*s.wx - s.qy*s.wy - s.qz*s.wz);
dq.x = 0.5 * ( s.qw*s.wx + s.qy*s.wz - s.qz*s.wy);
dq.y = 0.5 * ( s.qw*s.wy - s.qx*s.wz + s.qz*s.wx);
dq.z = 0.5 * ( s.qw*s.wz + s.qx*s.wy - s.qy*s.wx);
```

Pure unrolled Hamilton product, no symbolic quaternion type needed. The $\frac{1}{2}$ is right there in the code.

## Renormalization

Here's a thing that bites you in practice. After hundreds or thousands of integration steps, $\|\mathbf{q}\|$ slowly drifts away from $1$ due to floating point error. A non-unit quaternion does not just rotate, it also scales, and your simulator slowly explodes.

Fix: renormalize every step (or every few steps):

```cpp
double inv_norm = 1.0 / std::sqrt(q.w*q.w + q.x*q.x + q.y*q.y + q.z*q.z);
q.w *= inv_norm;
q.x *= inv_norm;
q.y *= inv_norm;
q.z *= inv_norm;
```

In Python with NumPy it's even cleaner:

```python
q = q / np.linalg.norm(q)
```

This is actually one of the practical reasons quaternions are nicer than rotation matrices. Renormalizing a quaternion is one sqrt and four multiplies. Renormalizing a $3 \times 3$ rotation matrix requires Gram-Schmidt orthogonalization, way more expensive and more error-prone.

## Where All of This Lives in My Simulator

Pulling it together, here is every place quaternions show up in the drone code:

**1. The state vector init:**

```cpp
state_ = {0,0,1, 0,0,0, 1,0,0,0, 0,0,0};
```

The middle four numbers are the identity quaternion, meaning the drone starts flat.

**2. World to body conversion using the conjugate:**

```cpp
qd_quat_rot(state_.qw, -state_.qx, -state_.qy, -state_.qz,
            0.0, 0.0, QD_GRAVITY, ax, ay, az);
```

This is the sandwich product with the conjugate of the orientation, rotating world gravity into body frame for the simulated accelerometer.

**3. The integration step inside RK4:**

```cpp
dq.w = 0.5 * (-s.qx*s.wx - s.qy*s.wy - s.qz*s.wz);
dq.x = 0.5 * ( s.qw*s.wx + s.qy*s.wz - s.qz*s.wy);
dq.y = 0.5 * ( s.qw*s.wy - s.qx*s.wz + s.qz*s.wx);
dq.z = 0.5 * ( s.qw*s.wz + s.qx*s.wy - s.qy*s.wx);
```

The quaternion kinematic equation $\dot{\mathbf{q}} = \frac{1}{2}\mathbf{q}\boldsymbol{\omega}$ written out by hand.

**4. PID controller using quaternion-derived Euler angles for attitude:**

```python
def quat_to_euler(qw, qx, qy, qz):
    roll  = math.atan2(2*(qw*qx + qy*qz), 1 - 2*(qx*qx + qy*qy))
    pitch = math.asin(clamp(2*(qw*qy - qz*qx), -1.0, 1.0))
    yaw   = math.atan2(2*(qw*qz + qx*qy), 1 - 2*(qy*qy + qz*qz))
    return roll, pitch, yaw
```

The PID inner loop works in Euler angles for the control law, but the underlying state is the quaternion. We convert just for the controller. The MEKF on the other hand stays purely in quaternion land.

## Wrap Up

Big post. So here is the TLDR:

- **The ladder of numbers**: $\mathbb{R}$ handles 1D, $\mathbb{C}$ gives a clean way to handle 2D rotation, and quaternions give a clean 4-number way to handle 3D rotation. A nice 3D-number algebra with the same multiplication and inverse behavior does not exist, so the useful next stop is 4D.
- A **unit quaternion** encodes a rotation as $(\cos\frac{\theta}{2}, \sin\frac{\theta}{2}\,\hat{\mathbf{n}})$, an axis and a half-angle.
- The **half angle** shows up because the sandwich product produces double-angle trig terms, so encoding $\theta/2$ gives a final rotation by $\theta$.
- **Rotating a vector**: sandwich product $\mathbf{q}\mathbf{v}_q\mathbf{q}^{\ast}$. Derivation collapses to Rodrigues' rotation formula.
- **For unit quaternions**, $\mathbf{q}^{-1} = \mathbf{q}^{\ast}$. This is the trick we used for world-to-body gravity rotation last post.
- **Integration**: $\dot{\mathbf{q}} = \frac{1}{2}\,\mathbf{q}\,\boldsymbol{\omega}_q$. Plug it into RK4 and you get smooth attitude updates.
- **Renormalize** every step or so, because floating point drift will scale your quaternion away from unit length.

Next post is probably going to be on the PID controller or the MEKF, both of which lean heavily on the quaternion stuff we just covered.

### Further Reading

If you want to go deeper:

- [3Blue1Brown, Visualizing Quaternions](https://www.youtube.com/watch?v=d4EgbgTm0Bg), the video everyone recommends and rightly so. Watch it twice.
- [3Blue1Brown, Quaternions and 3D Rotation, Explained Interactively](https://eater.net/quaternions), companion interactive site by Ben Eater and Grant Sanderson. Slide bars around and watch quaternions rotate things.
- [Joan Sola, Quaternion Kinematics for the Error-State Kalman Filter](https://arxiv.org/abs/1711.02508), the gold standard reference if you want to write your own MEKF later. Heavy but thorough.
- [ETH Zurich, An Introduction to 3D Orientations and Quaternions](https://ethz.ch/content/dam/ethz/special-interest/mavt/robotics-n-intelligent-systems/asl-dam/documents/lectures/robot_dynamics/RD2_Quaternions.pdf), a free PDF from the ETH robot dynamics course. Derivation-heavy and written for a robotics audience, so it will feel familiar after this post.

**Thanks for reading, this was a long one and the math is dense, so if I got anything wrong pls ping me on [X](https://x.com/JeetRex) and I will fix it.**
