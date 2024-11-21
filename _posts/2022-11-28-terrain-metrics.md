---
layout: post
title: Terrain metrics, by hand in R
date: 2022-12-13 12:00:00
description: Slope and curvature by the Evans-Young method

bibliography: 2022-11-28-terrain-metrics.bib 
---

Let's talk about terrain metrics. For all the time I have spent thinking about slope, relief, curvature, etc., I realized that I had never sat down and calculated these metrics from a DEM by hand. But, it is important to know why the terrain metrics tools we use in ArcGIs/QGIS/Earth Engine/whatever work. So, here is an R approach to calculating terrain metrics by hand. 

For much of this post, I'm following along with the description of the Evans-Young method given in [Florinsky 2012](https://www.sciencedirect.com/book/9780128046326/digital-terrain-analysis-in-soil-science-and-geology), but using the notation in the original: [Young and Evans 1978](https://www.researchgate.net/publication/279414703) Most GIS applications and software packages use this technique.

# The Evans-Young method

Consider a DEM $$Z$$ with regular grid spacing $$w$$ with each cell containing an elevation value $$z$$. Let us focus on a 3x3 window around a central cell. The eight cells adjacent to the central cell in this window is the eight-neighborhood of the central cell. We will refer to the elevation values in the eight-neighborhood as follows:

$$
\begin{bmatrix}
z_1 & z_2 & z_3 \\
z_4 & z_5 & z_6 \\
z_7 & z_8 & z_9 \\
\end{bmatrix}
$$

The coordinates of each elevation value in the eight neighborhood are:

$$
\begin{bmatrix}
(-w, w) & (0, w) & (w, w) \\
(-w, 0) & (0, 0) & (w, 0) \\
(-w, -w) & (0, -w) & (w, -w) \\
\end{bmatrix}
$$

The core of the Evans-Young method is to fit a second-degree polynomial to these elevation values, compute the partial derivatives of the polynomial, and then compute terrain metrics from the partial derivatives. The polynomial to be fit is:

$$
\begin{align*}
z = rx^2 + ty^2 + sxy + px + qy + u.
\end{align*}
$$

So, the partial derivatives correspond to the coefficients:

$$
\begin{align*}
p &= \frac{\partial z}{\partial x} \newline\newline
q &= \frac{\partial z}{\partial y} \newline\newline
2r &= \frac{\partial^2 z}{\partial x^2} \newline\newline
s &= \frac{\partial^2 z}{\partial x \partial y} \newline\newline
2t &= \frac{\partial^2 z}{\partial y^2}
\end{align*}
$$

The last coefficient, $$u$$, is not a partial derivative, but it is still a part of the fitting process. Now, we have to go about fitting the coefficients. To do this, we can make use of the fact that we can write values of $$z_i$$ as a system of equations. Using the indexing above, we can write that:

$$
\begin{align*}
z_1 &= w^2r + w^2t - w^2s - wp + wq + u \newline
z_2 &= 0 + w^2t + 0 + 0 + wq + u \newline
z_3 &= w^2r + w^2t + w^2s + wp + wq + u \newline
\vdots
\end{align*}
$$

This is a system of equations! To solve the system, we need to rewrite it in matrix form. Let $$\vec{z}$$ be a column vector of elevation values indexed as above and let the coefficient vector $$\vec{\alpha} = (r, t, s, p, q, u)^T$$. We can now write a matrix equation describing our eight-neighborhood as:

$$
\mathbf{F}\vec{\alpha} = \vec{z}
$$

$$\mathbf{F}$$ is the matrix of coefficients containing $$w$$ that we got when writing out the first few equations for $$z_i$$. The full matrix is:

$$
F = 
\begin{bmatrix}
w^2 & w^2 & -w^2 & -w & w  & 1 \\
0   & w^2 & 0    & 0  & w  & 1 \\
w^2 & w^2 & w^2  & w  & w  & 1 \\
w^2 & 0   & 0    & -w & 0  & 1 \\
0   & 0   & 0    & 0  & 0  & 1 \\
w^2 & 0   & 0    & w  & 0  & 1 \\
w^2 & w^2 & w^2  & -w & -w & 1 \\
0   & w^2 & 0    & 0  & -w & 1 \\
w^2 & w^2 & w^2  & w  & -w & 1
\end{bmatrix}
$$

Note that the $$i$$-th row of $$\mathbf{F}$$ corresponds to the equation for $$z_i$$. Now let's return to our matrix equation. To solve for $$\vec{\alpha}$$, we need to eliminate $$\mathbf{F}$$ from the left-hand side. We cannot take the inverse of $$\mathbf{F}$$ since it is not square. So, we will multiply both sides of the equation by $$\mathbf{F^T}$$ to create the square matrix $$\mathbf{F^T F}$$. Now we have:

$$
\mathbf{F^T F}\vec{\alpha} = \mathbf{F^T}\vec{z}
$$

Finally, multiply both sides by the inverse of the square matrix we just created: 

$$
(\mathbf{F^T F})^{-1}\mathbf{F^T F}\vec{\alpha} = (\mathbf{F^T F})^{-1}\mathbf{F^T}\vec{z}
$$

Since the left-hand side contains a matrix multiplied by its inverse, we are left with a 6x6 identity matrix that cancels out, giving us our solution:

$$
\vec{\alpha} = (\mathbf{F^T F})^{-1}\mathbf{F^T}\vec{z}
$$

On the right-hand side, we have:

$$
(\mathbf{F^T F})^{-1}\mathbf{F^T} =
\begin{bmatrix}
\frac{1}{6w^2} & \frac{-1}{3w^2} & \frac{1}{6w^2} & \frac{1}{6w^2} & \frac{-1}{3w^2} & \frac{1}{6w^2} & \frac{1}{6w^2} & \frac{-1}{3w^2} & \frac{1}{6w^2} \\

\frac{1}{6w^2} & \frac{1}{6w^2} & \frac{1}{6w^2} & \frac{-1}{3w^2} & \frac{-1}{3w^2} & \frac{-1}{3w^2} & \frac{1}{6w^2} & \frac{1}{6w^2} & \frac{1}{6w^2} \\

\frac{-1}{4w^2} & 0 & \frac{1}{4w^2} & 0 & 0 & 0 & \frac{1}{4w^2} & 0 & \frac{-1}{4w^2} \\

\frac{-1}{6w} & 0 & \frac{1}{6w} & \frac{-1}{6w} & 0 & \frac{1}{6w} & \frac{-1}{6w} & 0 & \frac{1}{6w} \\

\frac{1}{6w} & \frac{1}{6w} & \frac{1}{6w} & 0 & 0 & 0 & \frac{-1}{6w} & \frac{-1}{6w} & \frac{-1}{6w} \\

\frac{-1}{9} & \frac{2}{9} & \frac{-1}{9} & \frac{2}{9} & \frac{5}{9} & \frac{2}{9} & \frac{-1}{9} & \frac{2}{9} & \frac{-1}{9} \\
\end{bmatrix}
$$

(Believe me, I had about as much fun writing that in latex as you had reading it.)

Now we are in business. Take a closer look at the row corresponding to $$u$$. The vertical displacement of the polynomial is not just an average of all the elevation values! Each partial derivative is derived from the dot product of a row of the above matrix with the elevation vector $$\vec{z}$$. For example:

$$
\frac{\partial^2 z}{\partial x^2} = 2r = \frac{z_1 + z_3 + z_4 + z_6 + z_7 + z_9 - 2(z_2 + z_5 + z_8)}{3w^2}
$$

Before going any further, think about this equation in the context of the eight-neighborhood. The coefficients in front of the $$z_i$$'s match the below matrix.

$$
\begin{bmatrix}
1 & -2 & 1 \\
1 & -2 & 1 \\
1 & -2 & 1 \\
\end{bmatrix}
$$

So, you can think of the partial derivatives as *convolutions of the DEM*. We can then rewrite the partial derivative $$R$$ (capitalized because it applies to the entire image) as:

$$
R = Z \ast \begin{bmatrix} 1 & -2 & 1 \\ 1 & -2 & 1 \\ 1 & -2 & 1 \\ \end{bmatrix} / 3w^2

$$

All the other partials follow suit:

$$
\begin{align*}
P &= Z \ast \begin{bmatrix} -1 & 0 & 1 \\ -1 & 0 & 1 \\ -1 & 0 & 1 \\ \end{bmatrix} / 6w \newline
\newline
Q &= Z \ast \begin{bmatrix} 1 & 1 & 1 \\ 0 & 0 & 0 \\ -1 & -1 & -1 \\ \end{bmatrix} / 6w \newline
\newline
S &= Z \ast \begin{bmatrix} -1 & 0 & 1 \\ 0 & 0 & 0 \\ 1 & 0 & -1 \\ \end{bmatrix} / 4w^2 \newline
\newline
2T &= Z \ast \begin{bmatrix} 1 & 1 & 1 \\ -2 & -2 & -2 \\ 1 & 1 & 1 \\ \end{bmatrix} / 3w^2 \newline
\end{align*}
$$

As an aside, the fact that these partials come from convolutions makes them simple to calculate in Earth Engine. In fact, this is exactly how the [Terrain Analysis in Google Earth Engine](https://github.com/zecojls/tagee) package computes terrain metrics.

With our partials in hand, we can now calculate first and second-order terrain metrics. Slope is simply the gradient of the original DEM.

$$
G = \sqrt{p^2 + q^2}
$$

Note the similarity between $$G$$ and the gradient of the polynomial, $$\nabla z = \sqrt{(\frac{\partial z}{\partial x})^2 + (\frac{\partial z}{\partial y})^2}$$. These are the same property!

Horizontal and vertical curvature (aka [planform and profile curvature](https://www.esri.com/arcgis-blog/products/product/imagery/understanding-curvature-rasters/)) describe how slope changes in a landscape. They are used to describe the valley-ness or ridge-ness of a landform.

$$
\begin{align*}
k_h &= -\frac{2q^2r - 4pqr + 2p^2 t}{(p^2 + q^2)\sqrt{1+p^2+q^2}}
\newline \newline
k_v &= -\frac{2p^2r + 4pqs + 2q^2t}{(p^2+q^2)(1+p^2+q^2)^{1.5}}
\end{align*}
$$

# An R implementation

Let's play with some terrain metrics in R. We will start with an 8-neighborhood, visualize its Evans-Young polynomial, then calculate terrain metrics for the center cell. First, we will simply translate our convolutions described above into R. For those playing along at home, all the code for this post is in [this GitHub repository](https://github.com/s-kganz/terrain).

{% highlight r %}
get_evans_young_fit <- function(z, w=1) {
  r <- (z[1]+z[3]+z[4]+z[6]+z[7]+z[9] - 2*(z[2]+z[5]+z[8])) / (3*w^2)
  t <- (z[1]+z[2]+z[3]+z[7]+z[8]+z[9] - 2*(z[4]+z[5]+z[6])) / (3*w^2)
  s <- (z[3]+z[7]-z[1]-z[9]) / (4*w^2)
  p <- (z[3]+z[6]+z[9]-z[1]-z[4]-z[7]) / (6*w)
  q <- (z[1]+z[2]+z[3]-z[7]-z[8]-z[9]) / (6*w)
  u <- 1/9 * (2*(z[2]+z[4]+z[6]+z[8]) - (z[1]+z[3]+z[7]+z[9]) + 5*z[5])
  
  list(
    r=r, t=t, s=s, p=p, q=q, u=u
  )
}
{% endhighlight %}

Now, we use the `plotly` library to render the resulting polynomial.

{% highlight r %}
library(plotly)
render_evans_young_fit <- function(z, w=1) {
  partials <- get_evans_young_fit(z, w)
  r <- partials$r
  t <- partials$t
  s <- partials$s
  p <- partials$p
  q <- partials$q
  u <- partials$u
  
  # The valid range of the polynomial is *always* [-1, 1] for both x
  # and y axes.
  x_surf <- seq(-1, 1, length.out=100)
  y_surf <- seq(-1, 1, length.out=100)
  z_surf <- matrix(nrow=length(x_surf), ncol=length(y_surf))
  
  # This can be made much more efficient
  for (i in 1:length(x_surf)) {
    for (j in 1:length(y_surf)) {
      # Note flip of j and i here since y is our "row" on the surface.
      z_surf[j, i] <- r*x_surf[i]^2/2 + t*y_surf[j]^2/2 + s*x_surf[i]*y_surf[j] +
        p*x_surf[i] + q*y_surf[j] + u
    }
  }
  
  x_pts <- rep(c(-1, 0, 1), 3)
  y_pts <- c(rep(1, 3), rep(0, 3), rep(-1, 3))
  
  plot_ly() %>%
    add_surface(
      x=x_surf,
      y=y_surf,
      z=z_surf,
      opacity=0.5
    ) %>%
    add_markers(
      x=x_pts,
      y=y_pts,
      z=z
    )
}

z <- c(
    1, 4, 4,
    3, 3, 4,
    3, 4, 5
)

render_evans_young_fit(z)
{% endhighlight %}

<iframe width="100%" height="400" frameborder="0" scrolling="no" src="//plotly.com/~ganzk/52.embed"></iframe>


We can also calculate terrain metrics for this surface.

{% highlight r %}
get_terrain_metrics <- function(z, w=1) {
  partials <- get_evans_young_fit(z, w)
  r <- partials$r
  t <- partials$t
  s <- partials$s
  p <- partials$p
  q <- partials$q
  
  slope <- atan(sqrt(p^2+q^2)) # in rads
  
  # If p and q are both zero then the curvature result is 0/0, 
  # force to 0 since the surface is flat in this case.
  if (p == 0 & q == 0) {
    h_curv <- 0
    v_curv <- 0
  } else {
    h_curv <- -1 * (q^2*r - 2*p*q*s + p^2*t) /
      ( (p^2 + q^2) * sqrt(1+p^2+q^2) )
    v_curv <- -1 * (p^2*r + 2*p*q*s + q^2*t) /
      ( (p^2 + q^2) * (1+p^2+q^2)^1.5 )
  }
  
  relief <- max(abs(z[5] - z))
  grad   <- relief / w
  
  
  list(slope=slope, h_curv=h_curv, v_curv=v_curv,
       relief=relief, grad=grad)
}

> get_terrain_metrics(z1)
$slope
[1] 0.8410687

$h_curv
[1] -0.2222222

$v_curv
[1] 0.1975309

$relief
[1] 2

$grad
[1] 2
{% endhighlight %}

Note that this approach to calculating terrain metrics may differ from that in your favorite GIS software. Another popular technique is given in [Horn (1981)](doi.org/10.1109/PROC.1981.11918), and is currently in use in the R `raster` package.
